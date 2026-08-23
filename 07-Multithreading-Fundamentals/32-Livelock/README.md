# 7.32 — Livelock

## 🎯 Objective

Understand **livelock** in Java: how it differs from deadlock, how threads can remain active while making no useful progress, common causes, practical examples, prevention strategies, and interview scenarios.

> **Golden rule:** In a livelock, threads are **not blocked**; they keep executing and reacting to each other, but the system makes little or no useful progress.

---

# 1. What Is Livelock? ⭐⭐⭐⭐⭐

A **livelock** occurs when two or more threads remain active but continuously change their state or retry an operation without making meaningful progress.

Classic pattern:

```text
Thread-1 → detects conflict → backs off
Thread-2 → detects conflict → backs off

Thread-1 → retries → conflict
Thread-2 → retries → conflict

        ↓
  Both remain ACTIVE
        ↓
   No useful progress
        ↓
      LIVELOCK
```

### Key point

```text
Deadlock → threads are stuck waiting
Livelock  → threads are active but stuck in repeated reactions
```

---

# 2. Real-World Analogy

Two people meet in a narrow corridor.

```text
Person A moves left
Person B moves left

A moves right
B moves right

A moves left
B moves left
```

Both are trying to be polite, but neither passes.

That is the basic idea of livelock:

> **Lots of activity, but no progress.**

---

# 3. Livelock vs Deadlock ⭐⭐⭐⭐⭐

| Livelock | Deadlock |
|---|---|
| Threads remain active | Threads are blocked/waiting |
| Threads repeatedly change state/retry | Threads wait for resources/locks |
| No useful progress | No progress |
| Often caused by excessive retry/backoff/coordination | Often caused by circular resource dependency |
| CPU may continue being consumed | Threads may spend time blocked |

### Memory trick

```text
Deadlock → "We are waiting."
Livelock  → "We are moving, but going nowhere."
```

---

# 4. Livelock vs Starvation ⭐⭐⭐⭐⭐

### Starvation

A thread repeatedly fails to get the resource or execution opportunity it needs.

### Livelock

Threads actively execute but repeatedly interfere/react to each other without completing useful work.

```text
Starvation → "I keep getting denied."
Livelock   → "We keep reacting to each other."
```

---

# 5. Classic Livelock Example ⭐⭐⭐⭐⭐

Two threads repeatedly give way to each other.

```java
public class LivelockDemo {

    static class Person {
        private final String name;
        private boolean moving;

        Person(String name) {
            this.name = name;
        }

        boolean isMoving() {
            return moving;
        }

        void setMoving(boolean moving) {
            this.moving = moving;
        }

        void stepAside(Person other) {
            if (moving && other.isMoving()) {
                System.out.println(name + " steps aside for " + other.name);
                moving = false;
            }
        }
    }

    public static void main(String[] args) {
        Person a = new Person("A");
        Person b = new Person("B");

        a.setMoving(true);
        b.setMoving(true);

        for (int i = 0; i < 10; i++) {
            a.stepAside(b);
            b.stepAside(a);

            a.setMoving(!a.isMoving());
            b.setMoving(!b.isMoving());
        }
    }
}
```

This illustrates the **idea** of repeated reactive behavior. Exact livelock behavior is timing/state dependent.

---

# 6. Better Practice Example — Retry Livelock ⭐⭐⭐⭐⭐

A more practical example is two workers repeatedly detecting a conflict and immediately retrying.

```java
public class RetryLivelockDemo {

    private static volatile boolean conflict = true;

    static void worker(String name) {
        int attempts = 0;

        while (conflict && attempts < 20) {
            attempts++;

            System.out.println(name + " detects conflict and retries");

            // Both workers immediately retry using the same strategy.
            Thread.yield();
        }

        System.out.println(name + " finished after " + attempts + " attempts");
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> worker("T1"));
        Thread t2 = new Thread(() -> worker("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Important

This is a **livelock-prone retry pattern**, not a guaranteed livelock. `Thread.yield()` is only a scheduler hint and does not guarantee a particular scheduling result.

---

# 7. Livelock with Lock Acquisition ⭐⭐⭐⭐⭐

A common pattern is:

```text
T1 acquires A
T1 tries B → fails
T1 releases A

T2 acquires B
T2 tries A → fails
T2 releases B

T1 retries
T2 retries

        ↓
Repeated release/retry cycle
        ↓
No useful progress
```

If both threads use exactly the same retry behavior and timing, they may repeatedly collide.

---

# 8. `tryLock()` Can Be Used Incorrectly ⭐⭐⭐⭐⭐

`tryLock()` can help avoid waiting forever, but careless retry logic can create livelock.

Dangerous pattern:

```java
while (true) {
    if (lockA.tryLock()) {
        try {
            if (lockB.tryLock()) {
                try {
                    // work
                    return;
                } finally {
                    lockB.unlock();
                }
            }
        } finally {
            lockA.unlock();
        }
    }
}
```

If two threads repeatedly acquire one lock, fail to acquire the other, release, and immediately retry with the same timing, they can keep colliding.

---

# 9. Practice Code — `tryLock()` Livelock-Prone Pattern ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class TryLockLivelockDemo {

    private static final ReentrantLock A = new ReentrantLock();
    private static final ReentrantLock B = new ReentrantLock();

    static void work(String name, ReentrantLock first, ReentrantLock second)
            throws InterruptedException {

        for (int attempt = 1; attempt <= 100; attempt++) {
            if (!first.tryLock()) {
                continue;
            }

            try {
                if (!second.tryLock()) {
                    System.out.println(name + " releases and retries");
                    continue;
                }

                try {
                    System.out.println(name + " completed work");
                    return;
                } finally {
                    second.unlock();
                }
            } finally {
                first.unlock();
            }
        }

        System.out.println(name + " stopped retrying");
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            try {
                work("T1", A, B);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread t2 = new Thread(() -> {
            try {
                work("T2", B, A);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

The bounded retry count prevents the example from retrying forever.

---

# 10. Why `tryLock()` Does Not Automatically Solve Concurrency Problems

`tryLock()` avoids indefinite blocking for a particular acquisition attempt.

But this design:

```text
try
 ↓
fail
 ↓
release
 ↓
retry immediately
 ↓
fail
 ↓
retry
```

can consume CPU without accomplishing useful work.

Therefore:

> **Avoiding blocking is not the same as guaranteeing progress.**

---

# 11. Randomized Backoff — Important Prevention Technique ⭐⭐⭐⭐⭐

If multiple threads repeatedly collide, make them wait for different/random durations before retrying.

```text
T1 → conflict → wait 10 ms
T2 → conflict → wait 37 ms

T1 retries first
T1 completes
T2 retries later
```

Randomized backoff reduces synchronized retries.

---

# 12. Practice Code — Randomized Backoff ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.locks.ReentrantLock;

public class BackoffLivelockPreventionDemo {

    private static final ReentrantLock A = new ReentrantLock();
    private static final ReentrantLock B = new ReentrantLock();

    static void work(String name,
                     ReentrantLock first,
                     ReentrantLock second) throws InterruptedException {

        for (int attempt = 1; attempt <= 100; attempt++) {
            if (!first.tryLock()) {
                backoff();
                continue;
            }

            try {
                if (!second.tryLock()) {
                    backoff();
                    continue;
                }

                try {
                    System.out.println(name + " completed work");
                    return;
                } finally {
                    second.unlock();
                }
            } finally {
                first.unlock();
            }
        }

        System.out.println(name + " could not complete");
    }

    private static void backoff() throws InterruptedException {
        int millis = ThreadLocalRandom.current().nextInt(10, 50);
        Thread.sleep(millis);
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            try {
                work("T1", A, B);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread t2 = new Thread(() -> {
            try {
                work("T2", B, A);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Why it helps

The retry timings are less likely to remain perfectly synchronized.

---

# 13. Deterministic Lock Ordering — Better When Possible ⭐⭐⭐⭐⭐

If multiple locks are required, a global ordering is often better than relying on retries.

Bad:

```text
T1 → A → B
T2 → B → A
```

Better:

```text
T1 → A → B
T2 → A → B
```

This avoids the circular dependency that can cause deadlock and also removes the need for repeated collision/retry in many designs.

---

# 14. Practice Code — Fixed Lock Ordering ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.locks.ReentrantLock;

public class OrderedLocksDemo {

    private static final ReentrantLock A = new ReentrantLock();
    private static final ReentrantLock B = new ReentrantLock();

    static void work(String name) {
        A.lock();
        try {
            B.lock();
            try {
                System.out.println(name + " completed work");
            } finally {
                B.unlock();
            }
        } finally {
            A.unlock();
        }
    }

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> work("T1"));
        Thread t2 = new Thread(() -> work("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

Both threads use:

```text
A → B
```

so the lock dependency is consistent.

---

# 15. Idempotent Retry Design ⭐⭐⭐⭐

Retries are common in distributed systems and concurrent code.

If an operation is retried, design it so repeating the attempt does not produce unintended side effects.

For example:

```text
attempt
 ↓
conflict
 ↓
backoff
 ↓
retry safely
```

This is particularly important when concurrency control and external operations are combined.

---

# 16. Avoid Blind Infinite Retry ⭐⭐⭐⭐⭐

Dangerous:

```java
while (true) {
    if (!tryOperation()) {
        continue;
    }
    break;
}
```

Better:

```java
for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    if (tryOperation()) {
        return;
    }

    backoff();
}

throw new IllegalStateException("Operation could not complete");
```

Benefits:

- bounded resource usage
- predictable failure behavior
- easier monitoring
- reduced livelock risk

---

# 17. Common Causes of Livelock ⭐⭐⭐⭐⭐

### 1. Immediate Retry

Threads retry instantly after every conflict.

### 2. Identical Retry Strategy

All threads use the same timing and logic.

### 3. Excessive `tryLock()` Loops

Repeated acquisition/release can create a collision cycle.

### 4. Overly Polite Coordination

Threads repeatedly give up their opportunity for one another.

### 5. Symmetric Algorithms

Identical actors can make identical decisions repeatedly.

### 6. Poor Backoff Strategy

No delay or identical fixed delay can keep retries synchronized.

---

# 18. Prevention Strategies ⭐⭐⭐⭐⭐

### Strategy 1 — Use Lock Ordering

Prefer deterministic lock acquisition order.

### Strategy 2 — Add Backoff

Use delay before retrying.

### Strategy 3 — Randomize Backoff

Avoid synchronized retries.

### Strategy 4 — Limit Retries

Use a maximum retry count.

### Strategy 5 — Change Strategy After Repeated Failure

Do not blindly repeat the same action.

### Strategy 6 — Reduce Contention

Reduce the number of threads competing for the same resources.

### Strategy 7 — Prefer Higher-Level Concurrency Utilities

Avoid unnecessary custom coordination logic.

---

# 19. Livelock and `Thread.yield()` ⭐⭐⭐⭐⭐

`Thread.yield()` is only a scheduler hint.

It does **not** guarantee:

```text
another thread will run
```

and it does not guarantee that livelock will be prevented.

Therefore this is not a reliable livelock solution:

```java
while (conflict) {
    Thread.yield();
}
```

A real strategy requires a change in coordination/retry behavior.

---

# 20. Livelock and `sleep()` ⭐⭐⭐⭐

A fixed sleep can reduce collision probability, but identical fixed delays can still keep threads synchronized.

For example:

```java
Thread.sleep(100);
```

used by every competing thread may still result in repeated synchronized retries.

Randomized/exponential backoff is often more effective for retry-heavy systems.

---

# 21. Livelock and Exponential Backoff ⭐⭐⭐⭐⭐

A common retry strategy is to increase the delay after repeated conflicts.

```text
Attempt 1 → 10 ms
Attempt 2 → 20 ms
Attempt 3 → 40 ms
Attempt 4 → 80 ms
...
```

Usually a maximum delay is imposed.

Random jitter can be added:

```text
backoff = base delay + random jitter
```

This reduces synchronized retries.

---

# 22. Practice Code — Exponential Backoff with Jitter ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ThreadLocalRandom;

public class ExponentialBackoffDemo {

    private static final int MAX_RETRIES = 6;

    static void retryOperation() throws InterruptedException {
        long baseDelay = 20;

        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {

            if (tryOperation()) {
                System.out.println("Operation succeeded");
                return;
            }

            long exponentialDelay =
                    baseDelay * (1L << attempt);

            long jitter =
                    ThreadLocalRandom.current().nextLong(20);

            long delay = exponentialDelay + jitter;

            System.out.println("Retrying after " + delay + " ms");
            Thread.sleep(delay);
        }

        throw new IllegalStateException("Maximum retries exceeded");
    }

    static boolean tryOperation() {
        return false; // Practice simulation
    }

    public static void main(String[] args) throws Exception {
        try {
            retryOperation();
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

This pattern is useful for reducing synchronized retry behavior.

---

# 23. Production Diagnosis ⭐⭐⭐⭐⭐

A livelock may appear as:

```text
CPU usage → high or continuously active
Threads    → RUNNABLE / executing
Requests   → little or no useful completion
Logs       → repeated retry/conflict messages
```

Look for:

- repeated retries
- repeated lock acquisition failures
- identical retry intervals
- no successful completion
- high CPU without useful throughput
- repeated state transitions

### Diagnostic question

> "Are the threads blocked, or are they actively running but repeatedly undoing/retrying their work?"

That distinction is extremely useful when diagnosing deadlock vs livelock.

---

# 24. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1

> Livelock means threads are blocked.

❌ False.

The key property is that threads remain active.

### Trap 2

> `tryLock()` always prevents deadlock and livelock.

❌ False.

It can avoid indefinite blocking for a lock attempt, but repeated retry can create livelock.

### Trap 3

> `Thread.yield()` solves livelock.

❌ False.

`yield()` is only a scheduler hint.

### Trap 4

> `sleep(100)` guarantees different retry timing.

❌ False.

All threads may wake around the same time.

### Trap 5

> Livelock and starvation are identical.

❌ False.

Starvation is repeated denial; livelock is active repeated behavior without useful progress.

### Trap 6

> A livelock always consumes 100% CPU.

❌ False.

CPU usage depends on the retry strategy and workload.

---

# 25. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is livelock?

A situation where threads remain active and repeatedly respond/retry but make no useful progress.

### Q2. Difference between deadlock and livelock?

Deadlock involves blocked/waiting threads in a dependency cycle; livelock involves active threads repeatedly changing/retrying without useful progress.

### Q3. How can `tryLock()` lead to livelock?

If threads repeatedly acquire one lock, fail to acquire another, release the first, and immediately retry using the same timing/strategy, they can keep colliding.

### Q4. How do you prevent livelock?

Use deterministic lock ordering, bounded retries, backoff, randomized jitter, reduced contention, and better coordination strategies.

### Q5. Does `yield()` solve livelock?

No. It is only a scheduling hint and provides no guarantee about which thread runs next.

### Q6. Does `sleep()` solve livelock?

Not necessarily. Identical fixed delays can keep retries synchronized. Randomized or exponential backoff is usually more robust for retry-heavy designs.

### Q7. What is exponential backoff?

A strategy that increases the retry delay after repeated failures, often with random jitter.

### Q8. How do you identify livelock in production?

Look for active threads, repeated retries/conflicts, high CPU or continuous activity, and very low useful throughput.

### Q9. Can livelock happen without locks?

Yes. Any repeated coordination/retry algorithm can livelock if participants continuously react without making progress.

### Q10. Is livelock deterministic?

Not necessarily. Thread scheduling and timing can make it intermittent.

---

# 26. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Livelock is a concurrency problem where threads remain active but repeatedly react to each other or retry an operation without making useful progress. The key difference from deadlock is that deadlocked threads are typically blocked waiting for resources, while livelocked threads continue executing. A common Java example is two threads using `tryLock()` on multiple locks: each acquires one lock, fails to acquire the other, releases its lock, and immediately retries. If both follow the same retry strategy and timing, they can repeatedly collide. We can reduce livelock using deterministic lock ordering, bounded retries, randomized or exponential backoff with jitter, and reduced contention. `Thread.yield()` is not a reliable solution because it is only a scheduler hint. In production, repeated retry logs, active threads and low useful throughput can indicate livelock."**

---

# 27. Quick Revision ⭐⭐⭐⭐⭐

```text
LIVELOCK
   ↓
Threads are ACTIVE
   ↓
Keep reacting / retrying
   ↓
No useful progress

Example:
T1 → A → fail B → release → retry
T2 → B → fail A → release → retry

Prevention:
→ Lock ordering
→ Backoff
→ Random jitter
→ Exponential backoff
→ Bounded retries
→ Reduce contention

Remember:
Deadlock → blocked
Starvation → repeatedly denied
Livelock → active but no progress
```

### One-line memory trick

> **Livelock = "Everyone is moving, but nobody is getting anywhere."**

---

# 28. Practice Checklist

- [x] Livelock definition
- [x] Real-world analogy
- [x] Livelock vs deadlock
- [x] Livelock vs starvation
- [x] Retry livelock
- [x] `tryLock()` livelock-prone design
- [x] Lock ordering
- [x] Randomized backoff
- [x] Exponential backoff
- [x] Jitter
- [x] Bounded retries
- [x] `yield()` limitation
- [x] `sleep()` limitation
- [x] Production diagnosis
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.31 — Starvation](../31-Starvation/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.33 — Thread Communication Patterns**