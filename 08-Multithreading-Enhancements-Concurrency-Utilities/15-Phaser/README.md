# 8.15 — `Phaser`

> **Goal:** Understand `Phaser` as a reusable, flexible synchronization barrier for multiple threads/parties where the number of participants can change dynamically.

---

## 1. What is `Phaser`? ⭐⭐⭐⭐⭐

`Phaser` is a Java concurrency utility from `java.util.concurrent` used to synchronize threads across **multiple phases**.

Unlike a fixed `CyclicBarrier`, a `Phaser` allows parties to **register and deregister dynamically**.

```text
Phase 0
A ──┐
B ──┼── await advance ──→ Phase 1
C ──┘

Phase 1
A ──┐
B ──┼── await advance ──→ Phase 2
D ──┘
```

### Memory Trick

> **Phaser = reusable barrier + phases + dynamic registration.**

---

# 2. Why `Phaser`? ⭐⭐⭐⭐⭐

Use `Phaser` when:

- Work happens in multiple phases.
- Threads must synchronize at the end of each phase.
- The number of participating threads can change.
- Threads may join or leave during execution.
- You need more flexibility than `CyclicBarrier`.

Typical examples:

- Multi-stage processing
- Parallel batch processing
- Simulation rounds
- Workflow phases
- Testing/lifecycle coordination
- Recursive or dynamic task coordination

---

# 3. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class BasicPhaserExample {

    public static void main(String[] args) {

        Phaser phaser = new Phaser(3);

        Runnable task = () -> {
            String name = Thread.currentThread().getName();

            System.out.println(name + " completed phase 1");
            phaser.arriveAndAwaitAdvance();

            System.out.println(name + " completed phase 2");
            phaser.arriveAndAwaitAdvance();

            System.out.println(name + " finished");
            phaser.arriveAndDeregister();
        };

        new Thread(task, "Worker-1").start();
        new Thread(task, "Worker-2").start();
        new Thread(task, "Worker-3").start();
    }
}
```

### Flow

```text
Register 3 parties
       ↓
Phase 0
       ↓
All arrive
       ↓
Phase advances
       ↓
Phase 1
       ↓
All arrive
       ↓
Phase advances
       ↓
Workers deregister
```

---

# 4. Phaser Terminology ⭐⭐⭐⭐⭐

| Term | Meaning |
|---|---|
| Party | A participant registered with the phaser |
| Phase | Current synchronization round |
| Registration | Adding a party |
| Arrival | A party reaches the current phase barrier |
| Deregistration | Removing a party |
| Termination | Phaser stops advancing |

### Important

A **party is not necessarily the same thing as a Java thread**.

A thread can register one or more parties depending on the design.

---

# 5. Creating a Phaser ⭐⭐⭐⭐⭐

No registered parties:

```java
Phaser phaser = new Phaser();
```

With initial parties:

```java
Phaser phaser = new Phaser(3);
```

With a parent phaser:

```java
Phaser phaser = new Phaser(parent);
```

For most basic applications, the first two forms are the important ones.

---

# 6. `register()` ⭐⭐⭐⭐⭐

Adds one new party.

```java
phaser.register();
```

Example:

```java
Phaser phaser = new Phaser();

phaser.register();
phaser.register();

System.out.println(phaser.getRegisteredParties());
```

Output:

```text
2
```

---

# 7. `bulkRegister()` ⭐⭐⭐⭐

Registers multiple parties at once.

```java
phaser.bulkRegister(5);
```

Equivalent conceptually to registering five parties.

Useful when the number of participants is known at startup.

---

# 8. `arriveAndAwaitAdvance()` ⭐⭐⭐⭐⭐

This is one of the most important methods.

```java
phaser.arriveAndAwaitAdvance();
```

It means:

> "I have finished this phase. Wait until the phase advances."

```text
Worker A ── arrive ──┐
Worker B ── arrive ──┼── Phase advances
Worker C ── arrive ──┘
```

---

# 9. `arrive()` ⭐⭐⭐⭐⭐

Signals arrival without waiting for the next phase.

```java
int phase = phaser.arrive();
```

Useful when the current thread does not need to block after reporting completion.

---

# 10. `arriveAndDeregister()` ⭐⭐⭐⭐⭐

Signals arrival and permanently removes that party.

```java
phaser.arriveAndDeregister();
```

This is important when a participant has completed all of its work.

```text
Before:
A B C D

D finishes

After:
A B C
```

---

# 11. `arriveAndAwaitAdvance()` vs `arriveAndDeregister()` ⭐⭐⭐⭐⭐

| Method | Arrive | Wait | Remain registered |
|---|---|---|---|
| `arrive()` | Yes | No | Yes |
| `arriveAndAwaitAdvance()` | Yes | Yes | Yes |
| `arriveAndDeregister()` | Yes | No | No |

### Memory Trick

```text
arrive()
→ I'm done, but keep me registered.

arriveAndAwaitAdvance()
→ I'm done, wait for everyone.

arriveAndDeregister()
→ I'm done permanently; remove me.
```

---

# 12. `awaitAdvance()` ⭐⭐⭐⭐

Waits for the phaser to advance from a given phase.

```java
int phase = phaser.arrive();
phaser.awaitAdvance(phase);
```

Unlike `arriveAndAwaitAdvance()`, the caller is not necessarily the party responsible for arriving at that point.

---

# 13. Current Phase ⭐⭐⭐⭐

```java
int phase = phaser.getPhase();
```

Example:

```java
System.out.println("Current phase: " + phaser.getPhase());
```

The phase number generally starts at `0` and increments as phases advance.

---

# 14. Registered Parties ⭐⭐⭐⭐

```java
phaser.getRegisteredParties();
```

Returns the number of currently registered parties.

---

# 15. Arrived Parties ⭐⭐⭐⭐

```java
phaser.getArrivedParties();
```

Returns the number of parties that have arrived at the current phase.

---

# 16. Unarrived Parties ⭐⭐⭐⭐

```java
phaser.getUnarrivedParties();
```

Returns the number of parties that have not yet arrived at the current phase.

Relationship:

```text
Registered = Arrived + Unarrived
```

---

# 17. Practice — Three Processing Phases ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class MultiPhaseExample {

    public static void main(String[] args) {

        Phaser phaser = new Phaser(3);

        Runnable task = () -> {
            String name = Thread.currentThread().getName();

            System.out.println(name + " Phase 1: Loading");
            phaser.arriveAndAwaitAdvance();

            System.out.println(name + " Phase 2: Processing");
            phaser.arriveAndAwaitAdvance();

            System.out.println(name + " Phase 3: Saving");
            phaser.arriveAndDeregister();
        };

        for (int i = 1; i <= 3; i++) {
            new Thread(task, "Worker-" + i).start();
        }
    }
}
```

### Important

No worker can move to Phase 2 until the required parties have arrived at Phase 1.

---

# 18. Dynamic Registration ⭐⭐⭐⭐⭐

This is one of the biggest differences from `CyclicBarrier`.

```java
Phaser phaser = new Phaser(2);

phaser.register();
```

The registered party count changes dynamically.

### Example

```java
Phaser phaser = new Phaser(2);

Thread worker1 = new Thread(() -> {
    phaser.arriveAndAwaitAdvance();
});

Thread worker2 = new Thread(() -> {
    phaser.arriveAndAwaitAdvance();
});

phaser.register();

worker1.start();
worker2.start();
```

The third party must also eventually arrive or deregister for the phase to advance.

---

# 19. Practice — Dynamic Worker Registration ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class DynamicRegistrationExample {

    public static void main(String[] args) throws InterruptedException {

        Phaser phaser = new Phaser(1); // main thread registered

        for (int i = 1; i <= 3; i++) {
            phaser.register();

            final int workerId = i;

            new Thread(() -> {
                try {
                    System.out.println("Worker " + workerId + " started");
                    Thread.sleep(workerId * 300L);

                    System.out.println("Worker " + workerId + " reached phase 0");
                    phaser.arriveAndAwaitAdvance();

                    System.out.println("Worker " + workerId + " finished");

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    phaser.arriveAndDeregister();
                }
            }).start();
        }

        System.out.println("Main waiting for workers...");
        phaser.arriveAndAwaitAdvance();
        System.out.println("All registered workers reached phase 0");
    }
}
```

### Design Point

The main thread starts as a party so it can control the first phase, then releases itself with `arriveAndAwaitAdvance()`.

---

# 20. Dynamic Deregistration ⭐⭐⭐⭐⭐

Suppose five workers are registered but one worker finishes permanently.

```java
phaser.arriveAndDeregister();
```

Now the remaining workers do not have to wait for that completed worker in future phases.

```text
Phase 1
A B C D E

E finishes
↓
Deregister E

Phase 2
A B C D
```

---

# 21. Practice — Worker Leaves Early ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class DynamicDeregistrationExample {

    public static void main(String[] args) {

        Phaser phaser = new Phaser(3);

        Thread worker1 = new Thread(() -> {
            System.out.println("Worker 1 phase 1");
            phaser.arriveAndAwaitAdvance();

            System.out.println("Worker 1 finished permanently");
            phaser.arriveAndDeregister();
        });

        Thread worker2 = new Thread(() -> {
            System.out.println("Worker 2 phase 1");
            phaser.arriveAndAwaitAdvance();

            System.out.println("Worker 2 phase 2");
            phaser.arriveAndDeregister();
        });

        Thread worker3 = new Thread(() -> {
            System.out.println("Worker 3 phase 1");
            phaser.arriveAndAwaitAdvance();

            System.out.println("Worker 3 phase 2");
            phaser.arriveAndDeregister();
        });

        worker1.start();
        worker2.start();
        worker3.start();
    }
}
```

---

# 22. `Phaser` Termination ⭐⭐⭐⭐⭐

A phaser can become **terminated**.

Check it using:

```java
phaser.isTerminated();
```

A phase value with the termination bit set is represented by a negative value when returned through certain phase methods.

The easiest application-level check is:

```java
if (phaser.isTerminated()) {
    System.out.println("Phaser terminated");
}
```

---

# 23. `onAdvance()` ⭐⭐⭐⭐⭐

`Phaser` can be subclassed and `onAdvance()` overridden to control when it terminates.

```java
import java.util.concurrent.Phaser;

public class CustomPhaser extends Phaser {

    private final int maxPhases;

    public CustomPhaser(int parties, int maxPhases) {
        super(parties);
        this.maxPhases = maxPhases;
    }

    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        return phase >= maxPhases - 1 || registeredParties == 0;
    }
}
```

### Meaning

Returning `true` means:

> Terminate the phaser after this phase.

Returning `false` means:

> Continue to the next phase.

---

# 24. Practice — Terminate After N Phases ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class LimitedPhaseExample {

    static class LimitedPhaser extends Phaser {

        private final int maxPhases;

        LimitedPhaser(int parties, int maxPhases) {
            super(parties);
            this.maxPhases = maxPhases;
        }

        @Override
        protected boolean onAdvance(int phase, int registeredParties) {
            System.out.println("Completed phase: " + phase);

            return phase + 1 >= maxPhases || registeredParties == 0;
        }
    }

    public static void main(String[] args) {

        LimitedPhaser phaser = new LimitedPhaser(2, 3);

        Runnable task = () -> {
            while (!phaser.isTerminated()) {
                String name = Thread.currentThread().getName();
                System.out.println(name + " working in phase " + phaser.getPhase());

                phaser.arriveAndAwaitAdvance();
            }
        };

        new Thread(task, "Worker-1").start();
        new Thread(task, "Worker-2").start();
    }
}
```

---

# 25. `forceTermination()` ⭐⭐⭐⭐

You can force the phaser to terminate:

```java
phaser.forceTermination();
```

Useful for cancellation, shutdown or unrecoverable conditions.

After termination:

```java
phaser.isTerminated(); // true
```

---

# 26. Practice — Forced Shutdown ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class ForceTerminationExample {

    public static void main(String[] args) throws InterruptedException {

        Phaser phaser = new Phaser(2);

        Thread worker = new Thread(() -> {
            while (!phaser.isTerminated()) {
                System.out.println("Worker phase: " + phaser.getPhase());
                phaser.arriveAndAwaitAdvance();
            }

            System.out.println("Worker detected termination");
        });

        worker.start();

        Thread.sleep(500);
        phaser.forceTermination();
    }
}
```

---

# 27. `Phaser` vs `CyclicBarrier` ⭐⭐⭐⭐⭐

| Feature | `Phaser` | `CyclicBarrier` |
|---|---|---|
| Multiple phases | Yes | Yes |
| Reusable | Yes | Yes |
| Dynamic registration | Yes | No |
| Dynamic deregistration | Yes | No direct equivalent |
| Number of parties | Can change | Fixed after creation |
| Phase number | Yes | No explicit phase API |
| `arriveAndAwaitAdvance()` | Yes | No |
| `await()` | No | Yes |
| Custom termination | `onAdvance()` | No equivalent |
| Best for | Dynamic multi-phase workflows | Fixed-party repeated barriers |

### Interview Memory

> **CyclicBarrier = fixed group. Phaser = dynamic group.**

---

# 28. `Phaser` vs `CountDownLatch` ⭐⭐⭐⭐⭐

| Feature | `Phaser` | `CountDownLatch` |
|---|---|---|
| Reusable | Yes | No |
| Multiple phases | Yes | No |
| Dynamic registration | Yes | No |
| Dynamic deregistration | Yes | No |
| Main concept | Phase synchronization | One-shot countdown |

### Memory Trick

```text
Latch
→ Wait once for count to reach zero.

Phaser
→ Repeat synchronization across phases.
```

---

# 29. `Phaser` vs `Semaphore` ⭐⭐⭐⭐⭐

| Feature | `Phaser` | `Semaphore` |
|---|---|---|
| Main purpose | Phase synchronization | Concurrency/resource limiting |
| Participants | Parties | Permit holders |
| Multiple phases | Yes | Not its purpose |
| Value exchange | No | No |
| Dynamic parties | Yes | Permits can be acquired/released |

---

# 30. Practice — Multi-Phase Batch Processing ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class BatchProcessingExample {

    public static void main(String[] args) {

        Phaser phaser = new Phaser(4);

        for (int i = 1; i <= 4; i++) {
            final int workerId = i;

            new Thread(() -> {
                String name = "Worker-" + workerId;

                System.out.println(name + " loading data");
                phaser.arriveAndAwaitAdvance();

                System.out.println(name + " validating data");
                phaser.arriveAndAwaitAdvance();

                System.out.println(name + " persisting data");
                phaser.arriveAndDeregister();
            }, "Worker-" + i).start();
        }
    }
}
```

### Real-world interpretation

```text
Phase 1 → Load
Phase 2 → Validate
Phase 3 → Persist
```

Every worker must finish one stage before the group moves to the next stage.

---

# 31. Practice — Inspect Phaser State ⭐⭐⭐⭐

```java
import java.util.concurrent.Phaser;

public class PhaserStateExample {

    public static void main(String[] args) {

        Phaser phaser = new Phaser(3);

        System.out.println("Phase: " + phaser.getPhase());
        System.out.println("Registered: " + phaser.getRegisteredParties());
        System.out.println("Arrived: " + phaser.getArrivedParties());
        System.out.println("Unarrived: " + phaser.getUnarrivedParties());
        System.out.println("Terminated: " + phaser.isTerminated());
    }
}
```

Expected initial state:

```text
Phase: 0
Registered: 3
Arrived: 0
Unarrived: 3
Terminated: false
```

---

# 32. Important Pitfall — Registered Party Never Arrives ⚠️ ⭐⭐⭐⭐⭐

Suppose:

```java
Phaser phaser = new Phaser(3);
```

but only two parties arrive:

```text
A → arrive
B → arrive
C → never arrives
```

The phase cannot advance because the phaser is still waiting for C.

### Production Lesson

Every registered party must eventually:

- arrive,
- deregister,
- or the phaser must be terminated.

---

# 33. Important Pitfall — Registering Too Early / Too Late ⚠️

Registration is part of the synchronization protocol.

If a task registers itself after another party has already passed the relevant phase, carefully design which phase the new party should participate in.

### Rule

> Dynamic registration gives flexibility, but the application must define exactly which phase a newly registered party belongs to.

---

# 34. Important Pitfall — Forgetting Deregistration ⚠️

If a worker permanently finishes but remains registered:

```java
// wrong for a permanently finished worker
phaser.arrive();
```

Prefer:

```java
phaser.arriveAndDeregister();
```

when that party will not participate in future phases.

Otherwise, future phases may wait for a party that no longer exists.

---

# 35. Important Pitfall — Exception Handling ⚠️

If a worker fails before reaching the phase barrier, the remaining parties can wait indefinitely.

A robust design should define failure handling, for example:

```java
try {
    doWork();
    phaser.arriveAndAwaitAdvance();
} catch (Exception e) {
    phaser.arriveAndDeregister();
}
```

The exact recovery strategy depends on whether the failed worker can safely leave the workflow.

---

# 36. Important Pitfall — `Phaser` Does Not Make Work Thread-Safe ⚠️

`Phaser` synchronizes **phase progression**.

It does not automatically protect shared mutable state.

```java
phaser.arriveAndAwaitAdvance();
```

does not replace:

```java
synchronized
Lock
Atomic*
ConcurrentHashMap
```

### Interview Point

> Phaser coordinates when threads proceed; it does not protect arbitrary shared data from races.

---

# 37. Happens-Before Insight ⭐⭐⭐⭐⭐

Successful phaser synchronization provides the necessary ordering/visibility relationship associated with the phase advancement.

For interview purposes:

> Work completed before a party arrives at a phase is ordered before work performed after another party observes the phase advancement.

Do not confuse this with Phaser automatically making every shared variable operation atomic.

---

# 38. Practice — Phaser With ExecutorService ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class PhaserExecutorExample {

    public static void main(String[] args) {

        int workers = 3;
        Phaser phaser = new Phaser(workers);
        ExecutorService executor = Executors.newFixedThreadPool(workers);

        for (int i = 1; i <= workers; i++) {
            final int id = i;

            executor.submit(() -> {
                try {
                    System.out.println("Worker " + id + " phase 1");
                    phaser.arriveAndAwaitAdvance();

                    System.out.println("Worker " + id + " phase 2");
                    phaser.arriveAndAwaitAdvance();

                    System.out.println("Worker " + id + " done");
                } finally {
                    phaser.arriveAndDeregister();
                }
            });
        }

        executor.shutdown();
    }
}
```

### Important

Do not register more parties than the executor can actually run if those parties are required to arrive concurrently. Otherwise, blocking tasks can consume all workers while other registered participants remain queued.

---

# 39. Interview Scenario — Dynamic Number of Workers ⭐⭐⭐⭐⭐

### Problem

You have a multi-stage batch system where workers can be added or removed while the batch is running.

### Better fit

```text
Phaser
```

because it supports:

```text
register()
bulkRegister()
arrive()
arriveAndAwaitAdvance()
arriveAndDeregister()
```

### Why not `CyclicBarrier`?

Because `CyclicBarrier` has a fixed number of parties.

---

# 40. Interview Scenario — Fixed 5 Threads, Repeated Rounds ⭐⭐⭐⭐

If exactly five threads always participate in every round, both `CyclicBarrier` and `Phaser` can work.

Prefer:

```text
CyclicBarrier
```

when the synchronization model is simple and fixed.

Prefer:

```text
Phaser
```

when you need phase-oriented APIs, dynamic registration/deregistration or custom termination behavior.

---

# 41. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `Phaser`?

> `Phaser` is a reusable synchronization barrier that supports multiple phases and dynamic registration/deregistration of participating parties.

### Q2. Why use `Phaser` instead of `CyclicBarrier`?

> Phaser is more flexible when the number of participants can change or when the application needs explicit phase management and dynamic registration/deregistration.

### Q3. Is `Phaser` reusable?

> Yes. It can coordinate multiple phases.

### Q4. What is a party?

> A registered participant in the phaser. A party does not have to map one-to-one to a Java thread.

### Q5. What does `arriveAndAwaitAdvance()` do?

> The calling party arrives at the current phase and waits until the phase advances.

### Q6. What does `arriveAndDeregister()` do?

> The party arrives and is removed from future phase participation.

### Q7. How do you dynamically add a participant?

> Use `register()` or `bulkRegister()`.

### Q8. How do you force termination?

> Call `forceTermination()`.

### Q9. What is `onAdvance()`?

> A protected hook that can be overridden to control whether the phaser terminates after a phase completes.

### Q10. What happens if a registered party never arrives?

> The phase may never advance unless that party deregisters or the phaser is terminated.

### Q11. Is `Phaser` a replacement for synchronization on shared data?

> No. It coordinates phase progression; it does not automatically make shared mutable state thread-safe.

### Q12. `Phaser` vs `CountDownLatch`?

> CountDownLatch is normally a one-shot countdown; Phaser supports repeated phases and dynamic parties.

### Q13. `Phaser` vs `CyclicBarrier`?

> Both support reusable barriers, but Phaser supports dynamic registration/deregistration and richer phase control.

### Q14. Can the number of parties decrease?

> Yes, through `arriveAndDeregister()`.

### Q15. Can the number of parties increase?

> Yes, through `register()` or `bulkRegister()`.

---

# 42. Quick Revision

```text
Phaser
  ↓
Multiple phases
  ↓
Reusable synchronization
  ↓
Dynamic registration
  ↓
Dynamic deregistration
  ↓
Phase advances when required parties arrive
```

### Core APIs

```java
new Phaser()
new Phaser(parties)
register()
bulkRegister(n)
arrive()
arriveAndAwaitAdvance()
arriveAndDeregister()
awaitAdvance(phase)
getPhase()
getRegisteredParties()
getArrivedParties()
getUnarrivedParties()
isTerminated()
forceTermination()
onAdvance(...)
```

### Memory Trick ⭐

> **Phaser = Phase + Barrier + Dynamic Parties.**

---

# 🏆 2-Minute Interview Answer

> **"`Phaser` is a reusable synchronization utility from `java.util.concurrent` designed for applications that execute work in multiple phases. Threads, or more precisely parties, register with the phaser and arrive at the end of each phase. Methods such as `arriveAndAwaitAdvance()` allow a party to signal completion and wait for the phase to advance, while `arriveAndDeregister()` allows a party that has permanently finished to leave future phases. The biggest advantage over `CyclicBarrier` is dynamic registration and deregistration, along with explicit phase management and customizable termination through `onAdvance()`. A `CountDownLatch` is generally one-shot, whereas a Phaser is reusable across multiple phases. Phaser coordinates phase progression but does not by itself make shared mutable data thread-safe. In production, we must also handle failures carefully because a registered party that never arrives can prevent the phase from advancing."**

---

# 💻 Practice Checklist

- [ ] Create a `Phaser` with initial parties.
- [ ] Use `register()`.
- [ ] Use `bulkRegister()`.
- [ ] Practice `arrive()`.
- [ ] Practice `arriveAndAwaitAdvance()`.
- [ ] Practice `arriveAndDeregister()`.
- [ ] Practice multiple phases.
- [ ] Practice dynamic registration.
- [ ] Practice dynamic deregistration.
- [ ] Inspect phase/party counts.
- [ ] Practice `awaitAdvance()`.
- [ ] Practice `forceTermination()`.
- [ ] Override `onAdvance()`.
- [ ] Build a multi-stage batch processor.
- [ ] Integrate with `ExecutorService`.
- [ ] Understand executor capacity risks.
- [ ] Compare `Phaser` vs `CyclicBarrier`.
- [ ] Compare `Phaser` vs `CountDownLatch`.
- [ ] Compare `Phaser` vs `Semaphore`.
- [ ] Explain the registered-party failure scenario.
- [ ] Explain why Phaser does not make shared state thread-safe.
- [ ] Explain the topic in under 2 minutes.

---

## Navigation

[← 8.14 — `Exchanger`](../14-Exchanger/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.16 — `ReentrantLock`**