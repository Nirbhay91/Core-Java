# 7.33 — Thread Communication Patterns

## 🎯 Objective

Understand how Java threads coordinate work, wait for conditions, signal one another, and safely exchange data.

> **Golden rule:** Thread communication is about **coordination**, not merely sharing memory.

---

# 1. What Is Thread Communication? ⭐⭐⭐⭐⭐

Thread communication allows one thread to coordinate with another thread when work depends on a condition or event.

Classic example:

```text
Producer → produces data
Consumer → needs data

Consumer → waits when buffer is empty
Producer → signals that data is available
Consumer → wakes and consumes
```

Java provides several mechanisms:

- `wait()` / `notify()` / `notifyAll()`
- `BlockingQueue`
- `CountDownLatch`
- `CyclicBarrier`
- `Semaphore`
- `Future` / `CompletableFuture`
- `Condition`
- `Phaser`

---

# 2. `wait()` / `notify()` / `notifyAll()` ⭐⭐⭐⭐⭐

These are the classic intrinsic-monitor communication APIs.

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // perform work
}
```

Another thread can change the condition and signal:

```java
synchronized (lock) {
    condition = true;
    lock.notifyAll();
}
```

### Important

`wait()`:

- must be called while owning the object's monitor
- releases that monitor while waiting
- reacquires the monitor before returning

`notify()`:

- wakes one waiting thread, if one exists
- does not release the monitor immediately

`notifyAll()`:

- wakes all waiting threads
- they still compete to reacquire the monitor

---

# 3. Always Wait in a `while` Loop ⭐⭐⭐⭐⭐

Correct:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    useResource();
}
```

Avoid:

```java
synchronized (lock) {
    if (!condition) {
        lock.wait();
    }

    useResource();
}
```

Why?

A thread must re-check the condition after waking because another thread may consume/change the state before it reacquires the monitor.

---

# 4. Practice Code — Producer/Consumer with `wait()` / `notifyAll()` ⭐⭐⭐⭐⭐

```java
public class WaitNotifyProducerConsumer {

    private static final Object LOCK = new Object();
    private static Integer value;

    static void produce(int newValue) throws InterruptedException {
        synchronized (LOCK) {
            while (value != null) {
                LOCK.wait();
            }

            value = newValue;
            System.out.println("Produced: " + value);
            LOCK.notifyAll();
        }
    }

    static int consume() throws InterruptedException {
        synchronized (LOCK) {
            while (value == null) {
                LOCK.wait();
            }

            int result = value;
            value = null;
            System.out.println("Consumed: " + result);
            LOCK.notifyAll();
            return result;
        }
    }

    public static void main(String[] args) throws Exception {
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    produce(i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    consume();
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

# 5. `BlockingQueue` — Preferred Producer/Consumer Tool ⭐⭐⭐⭐⭐

Instead of manually managing:

```text
lock + condition + wait + notify
```

use a `BlockingQueue` when the problem is naturally a producer/consumer queue.

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
```

Producer:

```java
queue.put(value);
```

Consumer:

```java
Integer value = queue.take();
```

`put()` waits when the queue is full, and `take()` waits when it is empty.

---

# 6. Practice Code — `BlockingQueue` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BlockingQueueDemo {

    public static void main(String[] args) throws Exception {

        BlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(3);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    queue.put(i);
                    System.out.println("Produced: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    int value = queue.take();
                    System.out.println("Consumed: " + value);
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

### Interview point

For a standard producer/consumer queue, prefer `BlockingQueue` over manually implementing `wait()` / `notify()` unless the exercise specifically tests monitor mechanics.

---

# 7. `CountDownLatch` ⭐⭐⭐⭐⭐

`CountDownLatch` lets one or more threads wait until a count reaches zero.

```text
count = 3

Worker 1 → countDown()
Worker 2 → countDown()
Worker 3 → countDown()

count = 0
      ↓
waiting thread proceeds
```

Important property:

> A `CountDownLatch` is generally a **one-shot** coordination mechanism. Once its count reaches zero, it cannot be reset.

---

# 8. Practice Code — `CountDownLatch` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {

    public static void main(String[] args) throws Exception {

        CountDownLatch latch = new CountDownLatch(3);

        for (int i = 1; i <= 3; i++) {
            int workerId = i;

            new Thread(() -> {
                try {
                    Thread.sleep(workerId * 200L);
                    System.out.println("Worker " + workerId + " completed");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            }).start();
        }

        System.out.println("Main waiting...");
        latch.await();
        System.out.println("All workers completed");
    }
}
```

---

# 9. `CyclicBarrier` ⭐⭐⭐⭐⭐

`CyclicBarrier` allows a group of threads to wait until all participating threads reach a common barrier point.

```text
T1 → barrier ─┐
T2 → barrier ─┼→ all arrived → continue
T3 → barrier ─┘
```

Unlike `CountDownLatch`, a barrier can be reused after the parties cross it.

---

# 10. Practice Code — `CyclicBarrier` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {

    public static void main(String[] args) throws Exception {

        CyclicBarrier barrier = new CyclicBarrier(
                3,
                () -> System.out.println("All workers reached barrier")
        );

        for (int i = 1; i <= 3; i++) {
            int workerId = i;

            new Thread(() -> {
                try {
                    System.out.println("Worker " + workerId + " preparing");
                    Thread.sleep(workerId * 200L);

                    barrier.await();

                    System.out.println("Worker " + workerId + " continuing");
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

---

# 11. `Semaphore` ⭐⭐⭐⭐⭐

A `Semaphore` controls how many threads can access a resource concurrently.

```text
Permits = 2

T1 → acquire → allowed
T2 → acquire → allowed
T3 → acquire → waits
```

When a thread finishes:

```java
semaphore.release();
```

---

# 12. Practice Code — `Semaphore` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Semaphore;

public class SemaphoreDemo {

    private static final Semaphore SEMAPHORE =
            new Semaphore(2);

    static void accessResource(int id) {
        try {
            SEMAPHORE.acquire();

            System.out.println("Thread " + id + " acquired permit");
            Thread.sleep(500);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            SEMAPHORE.release();
            System.out.println("Thread " + id + " released permit");
        }
    }

    public static void main(String[] args) throws Exception {
        for (int i = 1; i <= 5; i++) {
            int id = i;
            new Thread(() -> accessResource(id)).start();
        }
    }
}
```

### Important

Only release a permit that your code actually acquired. In production code, track successful acquisition carefully if interruption/failure can occur before acquisition.

---

# 13. `Condition` with `ReentrantLock` ⭐⭐⭐⭐⭐

`Condition` provides monitor-style waiting/signalling associated with an explicit `Lock`.

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

Waiting:

```java
lock.lock();
try {
    while (!ready) {
        condition.await();
    }
} finally {
    lock.unlock();
}
```

Signalling:

```java
lock.lock();
try {
    ready = true;
    condition.signalAll();
} finally {
    lock.unlock();
}
```

---

# 14. Practice Code — `Condition` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionDemo {

    private static final ReentrantLock LOCK = new ReentrantLock();
    private static final Condition READY = LOCK.newCondition();
    private static boolean ready;

    static void awaitReady() throws InterruptedException {
        LOCK.lock();
        try {
            while (!ready) {
                READY.await();
            }

            System.out.println("Worker received signal");
        } finally {
            LOCK.unlock();
        }
    }

    static void makeReady() {
        LOCK.lock();
        try {
            ready = true;
            READY.signalAll();
        } finally {
            LOCK.unlock();
        }
    }

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                awaitReady();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        Thread.sleep(500);
        makeReady();

        worker.join();
    }
}
```

---

# 15. `Future` as a Result-Communication Mechanism ⭐⭐⭐⭐

A `Future` allows one thread to submit work and another thread to obtain its result.

```java
Future<Integer> future = executor.submit(() -> 100);

Integer result = future.get();
```

`get()` waits until the computation completes unless it is already complete or another completion/cancellation condition occurs.

### Important

A `Future` is primarily a **result/future-completion mechanism**, not a general-purpose condition-variable replacement.

---

# 16. Practice Code — `Future` ⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class FutureCommunicationDemo {

    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(500);
            return 42;
        });

        System.out.println("Waiting for result...");
        Integer result = future.get();
        System.out.println("Result = " + result);

        executor.shutdown();
    }
}
```

---

# 17. `CompletableFuture` ⭐⭐⭐⭐⭐

`CompletableFuture` supports asynchronous computation and composition.

Instead of manually coordinating:

```text
Thread A → produce result
Thread B → wait
Thread C → process result
```

we can express a pipeline:

```java
CompletableFuture
    .supplyAsync(this::loadData)
    .thenApply(this::transform)
    .thenAccept(this::save);
```

This is especially useful for asynchronous workflows.

---

# 18. Practice Code — `CompletableFuture` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureCommunicationDemo {

    public static void main(String[] args) {

        CompletableFuture<String> future =
                CompletableFuture
                        .supplyAsync(() -> "Java")
                        .thenApply(value -> value + " Multithreading")
                        .thenApply(String::toUpperCase);

        future.thenAccept(result ->
                System.out.println("Result = " + result));

        future.join();
    }
}
```

### Interview point

For asynchronous workflows, `CompletableFuture` can express dependencies between computations without manually coordinating raw threads.

---

# 19. `Phaser` ⭐⭐⭐⭐

`Phaser` supports reusable synchronization across multiple phases and a dynamic number of registered parties.

Conceptually:

```text
Phase 1 → all parties arrive
          ↓
Phase 2 → all parties arrive
          ↓
Phase 3 → all parties arrive
```

It is more flexible than a simple one-shot latch and can model phased algorithms.

---

# 20. Practice Code — `Phaser` ⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class PhaserDemo {

    public static void main(String[] args) throws Exception {

        Phaser phaser = new Phaser(3);

        for (int i = 1; i <= 3; i++) {
            int id = i;

            new Thread(() -> {
                try {
                    System.out.println("Worker " + id + " phase 1");
                    Thread.sleep(id * 100L);
                    phaser.arriveAndAwaitAdvance();

                    System.out.println("Worker " + id + " phase 2");
                    Thread.sleep(id * 100L);
                    phaser.arriveAndDeregister();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    phaser.arriveAndDeregister();
                }
            }).start();
        }
    }
}
```

---

# 21. Choosing the Right Communication Mechanism ⭐⭐⭐⭐⭐

| Requirement | Preferred Mechanism |
|---|---|
| Basic monitor coordination | `wait()` / `notify()` / `notifyAll()` |
| Producer/consumer queue | `BlockingQueue` |
| Wait for N tasks to finish | `CountDownLatch` |
| Reusable group barrier | `CyclicBarrier` |
| Limit concurrent access | `Semaphore` |
| Multiple condition queues with explicit lock | `Condition` |
| Obtain result of async task | `Future` |
| Compose async workflow | `CompletableFuture` |
| Multi-phase synchronization | `Phaser` |

### Interview rule

Do not automatically use `wait()` / `notify()` for every communication problem. First identify the coordination pattern, then choose the appropriate concurrency utility.

---

# 22. `notify()` vs `notifyAll()` ⭐⭐⭐⭐⭐

### `notify()`

Wakes one waiting thread.

Use only when the program's condition/state design makes waking one waiter sufficient and safe.

### `notifyAll()`

Wakes all waiting threads, which then compete to reacquire the monitor and re-check their conditions.

### Why `notifyAll()` is often safer

With multiple condition types or multiple consumers, waking an arbitrary single thread may wake a thread whose condition is still false.

```text
notify()
  ↓
wrong waiter wakes
  ↓
condition still false
  ↓
wait again
```

`notifyAll()` lets all waiters reconsider their predicates.

---

# 23. Practice Code — Multiple Conditions ⭐⭐⭐⭐⭐

```java
public class MultipleConditionDemo {

    private static final Object LOCK = new Object();
    private static boolean dataReady;
    private static boolean shutdown;

    static void awaitData() throws InterruptedException {
        synchronized (LOCK) {
            while (!dataReady && !shutdown) {
                LOCK.wait();
            }

            if (dataReady) {
                System.out.println("Data consumer proceeds");
            }
        }
    }

    static void shutdown() {
        synchronized (LOCK) {
            shutdown = true;
            LOCK.notifyAll();
        }
    }

    public static void main(String[] args) throws Exception {
        Thread consumer = new Thread(() -> {
            try {
                awaitData();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        consumer.start();
        Thread.sleep(300);
        shutdown();
        consumer.join();
    }
}
```

The consumer wakes and checks the condition again rather than assuming that waking means data is available.

---

# 24. Communication Is Different from Mutual Exclusion ⭐⭐⭐⭐⭐

### Mutual exclusion

Protects shared state:

```text
Only one thread enters critical section at a time.
```

Example:

```java
synchronized (lock) {
    counter++;
}
```

### Communication

Coordinates when a thread should wait or proceed:

```text
Producer → state changes
Consumer → waits for state
```

Example:

```java
while (!ready) {
    lock.wait();
}
```

A correct concurrent design may need both.

---

# 25. Visibility in Thread Communication ⭐⭐⭐⭐⭐

Communication requires that state changes become visible according to the Java Memory Model.

Synchronization establishes appropriate happens-before relationships.

For example:

```java
synchronized (lock) {
    ready = true;
    lock.notifyAll();
}
```

A waiting thread that subsequently reacquires the same monitor can observe the synchronized state correctly.

Do not replace proper synchronization with assumptions about CPU cache visibility.

---

# 26. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Calling `wait()` outside synchronized context

```java
lock.wait();
```

without owning `lock` can throw `IllegalMonitorStateException`.

### Mistake 2 — Using `if` instead of `while`

Always re-check the condition.

### Mistake 3 — Calling `notify()` and assuming the awakened thread runs immediately

It does not.

### Mistake 4 — Holding a lock unnecessarily during slow operations

Keep critical sections small.

### Mistake 5 — Using `wait()` / `notify()` when `BlockingQueue` fits naturally

Prefer higher-level concurrency utilities.

### Mistake 6 — Forgetting interruption handling

If an `InterruptedException` is caught and cannot be propagated, restoring the interrupt status is usually appropriate:

```java
Thread.currentThread().interrupt();
```

---

# 27. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is thread communication?

It is coordination between threads so that one thread can wait for a condition/event and another can signal or produce the required state.

### Q2. What are classic Java communication methods?

`wait()`, `notify()`, and `notifyAll()` using an object's monitor.

### Q3. Why use `while` around `wait()`?

Because after waking, the condition must be checked again before proceeding.

### Q4. Why is `BlockingQueue` preferred for producer/consumer?

It encapsulates thread-safe queueing and blocking behavior, reducing manual synchronization complexity.

### Q5. Difference between `CountDownLatch` and `CyclicBarrier`?

A latch is generally one-shot and lets waiting threads proceed when its count reaches zero. A barrier coordinates a fixed group reaching a common point and can be reused.

### Q6. What does `Semaphore` solve?

It limits concurrent access using permits.

### Q7. What is `Condition`?

A condition variable associated with a `Lock`, allowing threads to wait for specific predicates and signal waiting threads.

### Q8. When would you use `CompletableFuture`?

For asynchronous computations and composition of dependent stages.

### Q9. Why can `notify()` be dangerous with multiple conditions?

It may wake a thread whose condition is still false, leaving useful work waiting. The design must ensure correct condition handling.

### Q10. Does `notify()` release the lock?

No. The notifying thread keeps the monitor until it exits the synchronized region or otherwise releases the monitor.

### Q11. Does waking from `wait()` mean the thread immediately runs?

No. It must reacquire the monitor before returning from `wait()`, and scheduling is not guaranteed.

### Q12. What is the difference between communication and synchronization?

Synchronization primarily coordinates access/visibility and mutual exclusion; communication coordinates state-dependent progress between threads.

---

# 28. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Thread communication means coordinating concurrent threads based on shared state, conditions, results, or events. The classic Java mechanism is `wait()`/`notify()`/`notifyAll()` using an object's monitor, where the waiting thread checks a condition in a `while` loop and the producer changes the state and signals. For common production designs, higher-level utilities are usually preferable: `BlockingQueue` for producer-consumer, `CountDownLatch` for waiting for a set of tasks, `CyclicBarrier` for reusable group synchronization, `Semaphore` for limiting concurrent access, `Condition` for explicit lock-based conditions, and `CompletableFuture` for asynchronous result pipelines. The key is to choose the mechanism based on the coordination problem rather than manually implementing everything with `wait()` and `notify()`."**

---

# 29. Quick Revision ⭐⭐⭐⭐⭐

```text
THREAD COMMUNICATION
        ↓
Coordinate state / work / events
        ↓
Classic:
wait / notify / notifyAll
        ↓
Prefer higher-level tools where appropriate:

Producer/Consumer → BlockingQueue
Wait for N tasks  → CountDownLatch
Reusable barrier  → CyclicBarrier
Limit access      → Semaphore
Explicit conditions → Condition
Async result      → Future
Async pipeline    → CompletableFuture
Phased workflow   → Phaser
```

### Memory Trick

```text
Queue → BlockingQueue
Count → CountDownLatch
Group → CyclicBarrier
Permits → Semaphore
Condition → Condition
Result → Future
Pipeline → CompletableFuture
Phases → Phaser
```

---

# 30. Practice Checklist

- [x] Thread communication fundamentals
- [x] `wait()` / `notify()` / `notifyAll()`
- [x] `while` condition pattern
- [x] Producer/consumer with monitor
- [x] `BlockingQueue`
- [x] `CountDownLatch`
- [x] `CyclicBarrier`
- [x] `Semaphore`
- [x] `Condition`
- [x] `Future`
- [x] `CompletableFuture`
- [x] `Phaser`
- [x] Choosing the right mechanism
- [x] `notify()` vs `notifyAll()`
- [x] Communication vs mutual exclusion
- [x] Visibility considerations
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.32 — Livelock](../32-Livelock/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.34 — Common Thread-Safety Strategies**