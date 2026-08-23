# 8.12 — `CyclicBarrier`

> **Goal:** Understand how `CyclicBarrier` coordinates a fixed number of threads so that all participating parties reach the same synchronization point before any of them proceeds.

---

## 1. What is `CyclicBarrier`? ⭐⭐⭐⭐⭐

`CyclicBarrier` is a synchronization utility from `java.util.concurrent` used when multiple threads/parties must **meet at a common barrier** before continuing.

```text
Worker 1 ────────┐
Worker 2 ────────┤
Worker 3 ────────┤ → Barrier → all may continue
Worker 4 ────────┘
```

Each participating thread calls:

```java
barrier.await();
```

The barrier opens when the configured number of parties have arrived.

### Memory Trick ⭐

> **CyclicBarrier = all parties meet → barrier trips → all continue → barrier can be reused.**

---

# 2. Why `CyclicBarrier`? ⭐⭐⭐⭐⭐

Use it when work is divided into **phases** and all participating workers must finish the current phase before anyone starts the next phase.

Examples:

- Parallel algorithms
- Multi-stage simulations
- Batch processing phases
- Game rounds
- Distributed-style worker coordination inside one JVM
- Testing concurrent components

---

# 3. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CyclicBarrier;

public class BasicCyclicBarrierExample {

    public static void main(String[] args) {

        CyclicBarrier barrier = new CyclicBarrier(3);

        for (int i = 1; i <= 3; i++) {
            int workerId = i;

            new Thread(() -> {
                try {
                    System.out.println("Worker " + workerId + " reached barrier");

                    barrier.await();

                    System.out.println("Worker " + workerId + " passed barrier");
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

### Flow

```text
Barrier parties = 3

Worker 1 → await() → waiting
Worker 2 → await() → waiting
Worker 3 → await() → count reaches 3
                         ↓
                    Barrier trips
                    ↙    ↓    ↘
                 W1     W2     W3
                       continue
```

---

# 4. Constructor ⭐⭐⭐⭐

```java
CyclicBarrier barrier = new CyclicBarrier(4);
```

The number represents the number of parties required to trip the barrier.

It must be positive:

```java
new CyclicBarrier(0); // IllegalArgumentException
```

---

# 5. `await()` ⭐⭐⭐⭐⭐

A participating thread calls:

```java
barrier.await();
```

It waits until all required parties reach the barrier.

The method can throw:

```java
InterruptedException
BrokenBarrierException
```

There is also a timed form:

```java
barrier.await(5, TimeUnit.SECONDS);
```

The timed wait can additionally throw:

```java
TimeoutException
```

---

# 6. `await()` Return Value ⭐⭐⭐⭐

`await()` returns an arrival index:

```java
int arrivalIndex = barrier.await();
```

The last arriving party receives arrival index `0`.

Earlier arriving parties receive other non-negative indices.

### Important

Do **not** use the arrival index as a general ordering guarantee for application logic unless your design explicitly needs that property.

---

# 7. Barrier Action ⭐⭐⭐⭐⭐

`CyclicBarrier` can execute one action when the barrier trips.

```java
CyclicBarrier barrier = new CyclicBarrier(
        3,
        () -> System.out.println("All workers reached the barrier")
);
```

### Flow

```text
Worker 1 ─┐
Worker 2 ─┼─→ all arrive
Worker 3 ─┘
           ↓
     barrier action
           ↓
      workers continue
```

The barrier action runs before the parties are released from that barrier trip.

---

# 8. Practice — Barrier Action ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class BarrierActionExample {

    public static void main(String[] args) {

        CyclicBarrier barrier = new CyclicBarrier(
                3,
                () -> System.out.println("=== PHASE COMPLETE ===")
        );

        for (int i = 1; i <= 3; i++) {
            int id = i;

            new Thread(() -> {
                try {
                    System.out.println("Worker " + id + " completed phase 1");
                    barrier.await();
                    System.out.println("Worker " + id + " started phase 2");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (BrokenBarrierException e) {
                    System.out.println("Barrier broken");
                }
            }).start();
        }
    }
}
```

---

# 9. Why Is It Called **Cyclic**? ⭐⭐⭐⭐⭐

After all parties reach the barrier and it trips, the barrier can be used again.

```text
Round 1
W1 ─┐
W2 ─┼→ Barrier → continue
W3 ─┘

Round 2
W1 ─┐
W2 ─┼→ Barrier → continue
W3 ─┘

Round 3
...
```

This is the key difference from `CountDownLatch`.

---

# 10. Reusing the Barrier ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ReusableBarrierExample {

    public static void main(String[] args) {

        CyclicBarrier barrier = new CyclicBarrier(3);

        for (int i = 1; i <= 3; i++) {
            int id = i;

            new Thread(() -> {
                try {
                    for (int round = 1; round <= 3; round++) {
                        System.out.println(
                                "Worker " + id + " completed round " + round);

                        barrier.await();

                        System.out.println(
                                "Worker " + id + " starts next round");
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (BrokenBarrierException e) {
                    System.out.println("Barrier broken");
                }
            }).start();
        }
    }
}
```

---

# 11. Barrier vs Latch ⭐⭐⭐⭐⭐

| Feature | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reusable | ❌ No | ✅ Yes |
| Main idea | Wait for count to reach zero | Parties meet at a barrier |
| Common use | Task completion | Phase synchronization |
| Participants must call await | ❌ Not necessarily | ✅ Yes, for the barrier trip |
| Barrier action | ❌ | ✅ |
| Reset capability | ❌ | ✅ via `reset()` |

### Memory Trick

```text
Latch
→ "Wait until these tasks are complete."

Barrier
→ "Everyone reach this point before anyone continues."
```

---

# 12. Barrier vs `join()` ⭐⭐⭐⭐

`join()` normally waits for a specific thread to terminate:

```java
thread.join();
```

`CyclicBarrier` coordinates a group at a shared synchronization point:

```java
barrier.await();
```

### Key difference

```text
join()
→ lifecycle/completion of a particular thread

CyclicBarrier
→ phase synchronization among participating parties
```

---

# 13. Barrier vs `Semaphore` ⭐⭐⭐⭐

`Semaphore` controls the number of concurrent permits:

```text
N permits
   ↓
limit concurrent access
```

`CyclicBarrier` waits for a fixed group to reach a synchronization point:

```text
N parties
   ↓
all arrive
   ↓
continue
```

### Memory Trick

> **Semaphore = limit access. Barrier = synchronize arrival.**

---

# 14. Barrier vs `CountDownLatch` — Interview Scenario ⭐⭐⭐⭐⭐

### Scenario A

> Start 5 tasks and wait until all 5 complete.

Use:

```java
CountDownLatch
```

### Scenario B

> Five workers perform phase 1, then all must meet before phase 2. This repeats for many rounds.

Use:

```java
CyclicBarrier
```

---

# 15. `getNumberWaiting()` ⭐⭐⭐

You can inspect how many parties are currently waiting:

```java
int waiting = barrier.getNumberWaiting();
```

Example:

```java
System.out.println(
        "Waiting parties: " + barrier.getNumberWaiting());
```

This is useful for observation/debugging, but should not normally be used as the correctness mechanism for synchronization.

---

# 16. `getParties()` ⭐⭐⭐

Returns the configured number of parties:

```java
int parties = barrier.getParties();
```

Example:

```java
CyclicBarrier barrier = new CyclicBarrier(5);
System.out.println(barrier.getParties()); // 5
```

---

# 17. `isBroken()` ⭐⭐⭐⭐⭐

A barrier can enter the **broken** state.

Check it using:

```java
barrier.isBroken();
```

A barrier can become broken when a participating wait is interrupted, times out, or the barrier action fails.

---

# 18. `reset()` ⭐⭐⭐⭐⭐

A `CyclicBarrier` can be reset:

```java
barrier.reset();
```

This breaks the current barrier generation and allows the barrier to be used for a fresh generation.

### Important ⚠️

`reset()` is not something to call casually while threads are actively coordinating. Existing waiters may receive `BrokenBarrierException` and application-level coordination must account for that.

---

# 19. Broken Barrier ⭐⭐⭐⭐⭐

Consider:

```text
3 parties required

W1 → await()
W2 → await()
W3 → interrupted
```

The barrier can become broken and waiting parties can receive:

```java
BrokenBarrierException
```

### Practice Code

```java
import java.util.concurrent.*;

public class BrokenBarrierExample {

    public static void main(String[] args) throws InterruptedException {

        CyclicBarrier barrier = new CyclicBarrier(3);

        Thread worker1 = new Thread(() -> awaitBarrier(barrier), "worker-1");
        Thread worker2 = new Thread(() -> awaitBarrier(barrier), "worker-2");
        Thread worker3 = new Thread(() -> awaitBarrier(barrier), "worker-3");

        worker1.start();
        worker2.start();

        Thread.sleep(500);
        worker2.interrupt();

        worker1.join();
        worker2.join();
        worker3.start();
        worker3.join();
    }

    private static void awaitBarrier(CyclicBarrier barrier) {
        try {
            barrier.await();
            System.out.println(Thread.currentThread().getName() + " passed");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println(Thread.currentThread().getName() + " interrupted");
        } catch (BrokenBarrierException e) {
            System.out.println(Thread.currentThread().getName()
                    + " saw broken barrier");
        }
    }
}
```

> The exact scheduling/order of output is nondeterministic.

---

# 20. Timed `await()` ⭐⭐⭐⭐⭐

Always consider timeouts when barrier coordination depends on work that can stall.

```java
try {
    barrier.await(10, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    System.out.println("Barrier timed out");
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} catch (BrokenBarrierException e) {
    System.out.println("Barrier is broken");
}
```

A timeout can break the current barrier generation.

---

# 21. Practice — Timed Barrier ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class TimedBarrierExample {

    public static void main(String[] args) {

        CyclicBarrier barrier = new CyclicBarrier(3);

        for (int i = 1; i <= 3; i++) {
            int id = i;

            new Thread(() -> {
                try {
                    if (id == 1) {
                        Thread.sleep(3000);
                    }

                    System.out.println("Worker " + id + " waiting");
                    barrier.await(1, TimeUnit.SECONDS);
                    System.out.println("Worker " + id + " passed");

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (TimeoutException e) {
                    System.out.println("Worker " + id + " timed out");
                } catch (BrokenBarrierException e) {
                    System.out.println("Worker " + id + " saw broken barrier");
                }
            }).start();
        }
    }
}
```

---

# 22. Barrier Action Failure ⭐⭐⭐⭐⭐

Suppose the barrier action throws an exception:

```java
CyclicBarrier barrier = new CyclicBarrier(
        3,
        () -> {
            throw new RuntimeException("Phase failed");
        }
);
```

The barrier trip fails and the barrier becomes broken for that generation.

### Interview Point

> The barrier action is executed when the last party arrives, and if that action fails, the barrier generation is broken.

---

# 23. Practice — Multi-Phase Simulation ⭐⭐⭐⭐⭐

This is one of the most important practice programs.

```java
import java.util.concurrent.*;

public class MultiPhaseSimulation {

    static class Worker implements Runnable {

        private final int id;
        private final CyclicBarrier barrier;

        Worker(int id, CyclicBarrier barrier) {
            this.id = id;
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                for (int phase = 1; phase <= 3; phase++) {

                    System.out.println(
                            "Worker " + id
                                    + " completed phase " + phase);

                    barrier.await();

                    System.out.println(
                            "Worker " + id
                                    + " continues after phase " + phase);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } catch (BrokenBarrierException e) {
                System.out.println("Worker " + id + " stopped: barrier broken");
            }
        }
    }

    public static void main(String[] args) {

        int workers = 4;

        CyclicBarrier barrier = new CyclicBarrier(
                workers,
                () -> System.out.println("=== ALL WORKERS FINISHED PHASE ==="));

        ExecutorService executor = Executors.newFixedThreadPool(workers);

        for (int i = 1; i <= workers; i++) {
            executor.execute(new Worker(i, barrier));
        }

        executor.shutdown();
    }
}
```

### Mental Model

```text
Phase 1
W1 ─┐
W2 ─┤
W3 ─┤→ Barrier Action → Phase 2
W4 ─┘

Phase 2
W1 ─┐
W2 ─┤
W3 ─┤→ Barrier Action → Phase 3
W4 ─┘
```

---

# 24. Real-World Scenario — Batch Processing ⭐⭐⭐⭐⭐

Imagine four workers process a batch in stages:

```text
Stage 1: Read data
       ↓
Barrier
       ↓
Stage 2: Transform data
       ↓
Barrier
       ↓
Stage 3: Validate data
```

The barrier ensures no worker starts the next stage before every worker has completed the current stage.

### Important production considerations

- Workers should have a clear failure strategy.
- Avoid indefinite waits where possible.
- Consider task cancellation.
- Monitor barrier wait duration.
- Handle `BrokenBarrierException`.
- Ensure the number of participating parties matches the design.

---

# 25. Real-World Scenario — Game Rounds 🎮 ⭐⭐⭐⭐

For a game simulation:

```text
Player 1 → finish round
Player 2 → finish round
Player 3 → finish round
Player 4 → finish round
             ↓
         Barrier
             ↓
       Next round
```

The barrier can coordinate the start of the next round.

This is a conceptual example; real multiplayer systems normally require distributed coordination outside a single JVM.

---

# 26. `CyclicBarrier` with ExecutorService ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ExecutorBarrierExample {

    public static void main(String[] args) {

        int workers = 3;

        CyclicBarrier barrier = new CyclicBarrier(
                workers,
                () -> System.out.println("Batch phase complete"));

        ExecutorService executor = Executors.newFixedThreadPool(workers);

        for (int i = 1; i <= workers; i++) {
            int id = i;

            executor.submit(() -> {
                try {
                    System.out.println("Task " + id + " processing...");
                    Thread.sleep(id * 500L);

                    barrier.await();

                    System.out.println("Task " + id + " moving to next phase");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (BrokenBarrierException e) {
                    System.out.println("Task " + id + ": barrier broken");
                }
            });
        }

        executor.shutdown();
    }
}
```

---

# 27. Critical Pitfall — Wrong Party Count ⚠️

Suppose:

```java
CyclicBarrier barrier = new CyclicBarrier(4);
```

but only 3 workers ever call:

```java
barrier.await();
```

The three workers can remain waiting indefinitely unless a timeout, interruption, or reset breaks the generation.

### Memory Rule

> **Barrier parties must match the actual participants.**

---

# 28. Critical Pitfall — Thread Pool Too Small ⚠️ ⭐⭐⭐⭐⭐

A classic problem:

```java
CyclicBarrier barrier = new CyclicBarrier(4);
ExecutorService executor = Executors.newFixedThreadPool(2);
```

You submit four tasks and each task waits at the barrier.

```text
2 tasks run
 ↓
2 tasks reach barrier
 ↓
They wait
 ↓
Remaining 2 tasks cannot start
 ↓
Barrier never trips
```

This can cause a deadlock-like stall.

### Rule

> If all barrier parties must execute concurrently, the executor must provide enough worker capacity for the participating tasks, subject to the overall scheduling design.

---

# 29. Practice — Demonstrate Pool-Starvation Stall ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class BarrierPoolSizeExample {

    public static void main(String[] args) throws InterruptedException {

        int parties = 4;

        CyclicBarrier barrier = new CyclicBarrier(parties);
        ExecutorService executor = Executors.newFixedThreadPool(2);

        for (int i = 1; i <= parties; i++) {
            int id = i;

            executor.execute(() -> {
                try {
                    System.out.println("Task " + id + " reached barrier");
                    barrier.await();
                    System.out.println("Task " + id + " passed barrier");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (BrokenBarrierException e) {
                    System.out.println("Task " + id + " saw broken barrier");
                }
            });
        }

        executor.shutdown();
        executor.awaitTermination(2, TimeUnit.SECONDS);

        System.out.println("Barrier broken: " + barrier.isBroken());
    }
}
```

This illustrates why blocking synchronization inside a bounded executor requires careful capacity analysis.

---

# 30. `reset()` vs Creating a New Barrier ⭐⭐⭐⭐

`reset()` exists:

```java
barrier.reset();
```

But resetting an actively used barrier can disrupt waiting parties.

In many designs, creating a new barrier generation/object can be easier to reason about than forcing a reset.

### Interview point

> `reset()` is available because the barrier is reusable, but resetting during active coordination requires careful handling of the threads from the previous generation.

---

# 31. Thread Interruption ⭐⭐⭐⭐⭐

`await()` is interruptible.

Correct handling:

```java
try {
    barrier.await();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

Do not silently swallow the interruption.

If one participant is interrupted while waiting, the current barrier generation can become broken and other waiting participants may receive `BrokenBarrierException`.

---

# 32. Common Mistakes ❌

### Mistake 1 — Using `CountDownLatch` for repeated phases

Use `CyclicBarrier` when the synchronization point must be reused.

### Mistake 2 — Wrong number of parties

The barrier can wait indefinitely.

### Mistake 3 — Ignoring `BrokenBarrierException`

Barrier failure is part of the coordination protocol.

### Mistake 4 — Ignoring interruption

Restore the interrupt status or propagate it.

### Mistake 5 — Pool starvation

Too few executor threads can prevent all barrier parties from reaching the barrier.

### Mistake 6 — Assuming arrival order

Thread scheduling is nondeterministic.

### Mistake 7 — Heavy barrier action

The barrier action should not become an unnecessary bottleneck.

### Mistake 8 — Treating barrier as distributed coordination

`CyclicBarrier` coordinates threads within one JVM. It is not a distributed-system barrier.

---

# 33. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. What is `CyclicBarrier`?

> A reusable synchronization utility where a fixed number of parties wait at a common barrier before continuing.

### Q2. Why is it called cyclic?

> Because after the barrier trips, it can be reused for another synchronization cycle.

### Q3. What happens when all parties call `await()`?

> The barrier trips and the waiting parties are released to continue.

### Q4. Can a barrier be reused?

> Yes.

### Q5. Can a barrier be reset?

> Yes, using `reset()`, but resetting an active barrier requires careful coordination because the current generation can be broken.

### Q6. What is the barrier action?

> An optional `Runnable` executed when the final party arrives and the barrier trips.

### Q7. What is `BrokenBarrierException`?

> It indicates that the current barrier generation has been broken, for example because another waiting party was interrupted or timed out, or the barrier action failed.

### Q8. What happens if one party never reaches the barrier?

> Other parties can wait indefinitely unless the wait is interrupted, timed out, or the barrier is otherwise broken/reset.

### Q9. `CyclicBarrier` vs `CountDownLatch`?

> Latch is one-shot and commonly coordinates completion; barrier is reusable and coordinates participating parties at repeated synchronization points.

### Q10. `CyclicBarrier` vs `join()`?

> `join()` waits for a thread to terminate; `CyclicBarrier` coordinates a group at a phase boundary.

### Q11. `CyclicBarrier` vs `Semaphore`?

> Barrier synchronizes arrival; semaphore limits concurrent access through permits.

### Q12. What is a common executor problem with barriers?

> If the pool has fewer active worker threads than barrier parties, tasks can block at the barrier while remaining tasks cannot start, causing a pool-starvation stall.

### Q13. Is `await()` interruptible?

> Yes.

### Q14. What does `getNumberWaiting()` return?

> The approximate/current number of parties waiting at the barrier at that instant; it is mainly observational.

### Q15. What does `isBroken()` tell you?

> Whether the current barrier is in a broken state.

### Q16. Explain a multi-phase processing use case.

> Workers process phase 1, call `await()`, all workers synchronize, then they process phase 2, and the same barrier is reused.

---

# 34. Quick Revision

```text
CyclicBarrier(N)
       ↓
N parties
       ↓
await()
       ↓
all parties arrive
       ↓
barrier action (optional)
       ↓
all continue
       ↓
barrier reusable
```

### Key APIs

```java
new CyclicBarrier(N)
new CyclicBarrier(N, action)
barrier.await()
barrier.await(timeout, unit)
barrier.getParties()
barrier.getNumberWaiting()
barrier.isBroken()
barrier.reset()
```

### Key facts ⭐

```text
✓ Reusable
✓ Coordinates a fixed number of parties
✓ Supports barrier action
✓ Supports timed await
✓ await() is interruptible
✓ Can become broken
✓ Supports reset()
✓ Good for repeated phases
```

---

# 🎯 Interview Questions

1. What is `CyclicBarrier`?
2. Why is it called cyclic?
3. How does `await()` work?
4. What is a barrier action?
5. When is the barrier action executed?
6. Is `CyclicBarrier` reusable?
7. What does `reset()` do?
8. What is `BrokenBarrierException`?
9. What happens when one party is interrupted?
10. What happens when one party times out?
11. What if one party never reaches the barrier?
12. What is `getNumberWaiting()`?
13. What is `getParties()`?
14. What is `isBroken()`?
15. `CyclicBarrier` vs `CountDownLatch`?
16. `CyclicBarrier` vs `join()`?
17. `CyclicBarrier` vs `Semaphore`?
18. What is a start/phase synchronization pattern?
19. How can a small thread pool cause a barrier stall?
20. How would you coordinate multi-stage batch processing?
21. How would you handle barrier failure in production?
22. Explain `CyclicBarrier` in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"`CyclicBarrier` is a synchronization utility from `java.util.concurrent` used when a fixed number of participating threads must reach a common synchronization point before any of them proceeds. Each party calls `await()`. When the required number of parties arrive, the barrier trips, an optional barrier action can execute, and the parties continue. It is called cyclic because the same barrier can be reused for subsequent phases. Unlike `CountDownLatch`, which is one-shot and commonly represents completion signals, `CyclicBarrier` is designed for repeated phase synchronization. A barrier can become broken when a waiting party is interrupted or times out, or when the barrier action fails, so production code should handle `BrokenBarrierException`, interruption and timeouts. Another important pitfall is executor starvation: if the barrier expects four parties but only two executor threads can run the tasks, the first two may wait at the barrier while the other two cannot start."**

---

# 💻 Practice Checklist

- [ ] Create a basic `CyclicBarrier`.
- [ ] Use `await()`.
- [ ] Use a barrier action.
- [ ] Reuse the barrier across multiple phases.
- [ ] Use timed `await()`.
- [ ] Handle `InterruptedException`.
- [ ] Handle `BrokenBarrierException`.
- [ ] Check `isBroken()`.
- [ ] Inspect `getNumberWaiting()`.
- [ ] Use `getParties()`.
- [ ] Practice `reset()` carefully.
- [ ] Integrate with `ExecutorService`.
- [ ] Simulate a multi-phase batch process.
- [ ] Demonstrate wrong party count.
- [ ] Demonstrate executor pool-starvation stall.
- [ ] Compare with `CountDownLatch`.
- [ ] Compare with `Semaphore`.
- [ ] Explain the topic in under 2 minutes.

---

## Navigation

[← 8.11 — `CountDownLatch`](../11-CountDownLatch/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.13 — `Semaphore`**