# 7.9 — `join()`

## 🎯 Objective

Understand how `Thread.join()` allows one thread to **wait for another thread to complete** before continuing.

> **Memory trick:** `join()` = **wait for this thread to finish**.

---

## 1. What is `join()`? ⭐⭐⭐⭐⭐

Suppose the main thread starts a worker:

```java
worker.start();
```

Normally, main and worker can continue independently.

If main needs to wait until the worker finishes:

```java
worker.join();
```

Conceptually:

```text
main
  |
  | start worker
  ↓
worker ----------------→ finishes
  |
  | join()
  ↓
main continues
```

`join()` is therefore commonly used when one thread depends on another thread's completion.

---

## 2. Basic Example

```java
public class JoinDemo {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker started");

            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println("Worker finished");
        });

        worker.start();
        worker.join();

        System.out.println("Main continues");
    }
}
```

Expected order:

```text
Worker started
Worker finished
Main continues
```

The important point is that `main` does not continue past `join()` until `worker` has terminated, assuming the join is not interrupted.

---

## 3. What Does `join()` Actually Mean? ⭐⭐⭐⭐⭐

If you write:

```java
worker.join();
```

it means:

> The **current thread** waits for `worker` to terminate.

This is important because `join()` is called on the target thread, but the **caller/current thread** is the one that waits.

Example:

```text
main calls worker.join()
        ↓
main waits
        ↓
worker completes
        ↓
main continues
```

---

## 4. `join()` Does NOT Mean the Target Thread Joins the Caller

A common misunderstanding is:

```java
worker.join();
```

means worker joins main.

❌ Not in the synchronization sense.

The correct mental model is:

```text
CURRENT THREAD → waits for → TARGET THREAD
```

So if main executes:

```java
worker.join();
```

then:

```text
main waits for worker
```

---

## 5. `join()` and Thread Lifecycle ⭐⭐⭐⭐⭐

Consider:

```java
worker.start();
worker.join();
```

The worker progresses through its normal lifecycle and eventually reaches:

```text
TERMINATED
```

The caller waiting in `join()` can continue once the target thread has terminated.

Conceptually:

```text
worker:
NEW → RUNNABLE → ... → TERMINATED
                         ↑
                         |
                  join() completes
```

---

## 6. Why is `join()` Useful?

Typical use cases:

- Waiting for a background task to finish
- Making sure initialization completes before continuing
- Waiting for multiple worker threads
- Coordinating phases of a computation
- Simple thread orchestration in Core Java
- Interview demonstrations of thread dependency

Example:

```text
Main
 ├── start Task A
 ├── start Task B
 ├── join Task A
 ├── join Task B
 └── combine results
```

---

## 7. `join()` Is Interruptible ⭐⭐⭐⭐⭐

`join()` can throw:

```java
InterruptedException
```

Example:

```java
try {
    worker.join();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

If the thread currently waiting in `join()` is interrupted, the wait ends by throwing `InterruptedException`.

The target worker is **not automatically interrupted** merely because the caller's `join()` was interrupted.

---

## 8. Why Restore the Interrupt Flag?

A common pattern is:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

This preserves the interruption signal for higher-level code.

Another valid approach is to propagate `InterruptedException` if the surrounding method can declare it:

```java
public static void main(String[] args)
        throws InterruptedException {

    worker.join();
}
```

---

## 9. `join(long millis)`

Java also provides a timed version:

```java
worker.join(1000);
```

This means the current thread waits for the worker to terminate **for up to approximately 1000 milliseconds**.

It does **not** guarantee that the worker will terminate within that time.

If the timeout expires first, the waiting thread continues even if the worker is still alive.

---

## 10. `join(long millis, int nanos)`

There is also an overload:

```java
worker.join(1000, 500_000);
```

It allows an additional nanosecond component within the API's specified constraints.

As with other timed operations, this should not be treated as an exact scheduling guarantee.

---

## 11. `join()` vs `join(timeout)` ⭐⭐⭐⭐⭐

| `join()` | `join(timeout)` |
|---|---|
| Waits until target terminates | Waits up to specified timeout |
| No timeout | Has timeout |
| Caller continues after target terminates | Caller may continue because timeout expired |
| Can throw `InterruptedException` | Can throw `InterruptedException` |

---

## 12. `join()` Does Not Stop the Target Thread

This is important.

```java
worker.join();
```

does not:

- stop the worker
- pause the worker
- interrupt the worker
- kill the worker

It only causes the **current thread** to wait for the worker's termination.

```text
join()
  ↓
caller waits
  ↓
target keeps executing
  ↓
target terminates
  ↓
caller continues
```

---

## 13. `join()` vs `sleep()` ⭐⭐⭐⭐⭐

| `join()` | `sleep()` |
|---|---|
| Waits for another thread to terminate | Pauses current thread for a duration |
| Method of `Thread` | Static method of `Thread` |
| Target thread matters | Current thread matters |
| Duration may depend on target completion | Duration is requested explicitly |
| Can be timed | Can be timed |
| Interruptible | Interruptible |

### Memory trick

```text
sleep() → wait for TIME
join()  → wait for THREAD
```

---

## 14. `join()` vs `wait()` ⭐⭐⭐⭐⭐

These are different mechanisms.

| `join()` | `wait()` |
|---|---|
| Method of `Thread` | Method of `Object` |
| Used to wait for a thread to terminate | Used for coordination around an object's monitor |
| Caller waits for target thread | Caller waits for notification/timeout |
| Commonly used for thread completion | Commonly used for producer-consumer style coordination |
| Does not require caller to own target's monitor explicitly | Must own the object's monitor to call `wait()` |

---

## 15. Waiting for Multiple Threads ⭐⭐⭐⭐⭐

A common pattern:

```java
Thread t1 = ...;
Thread t2 = ...;
Thread t3 = ...;

t1.start();
t2.start();
t3.start();

t1.join();
t2.join();
t3.join();

System.out.println("All tasks completed");
```

The current thread waits for each target to terminate.

This is useful when the final operation depends on all workers completing.

---

## 16. Important Subtlety with Multiple `join()` Calls

Consider:

```java
t1.start();
t2.start();

t1.join();
t2.join();
```

Both workers were started before the joins.

Therefore `t1` and `t2` can execute concurrently.

The main thread simply waits for them one by one from its perspective.

```text
t1 ───────────────→ done
       \       
        \       
         t2 ─────────→ done

main: start both → wait t1 → wait t2 → continue
```

The joins do **not** make the worker threads execute sequentially.

---

## 17. Bad Pattern: Starting Before Every Join

Compare:

```java
t1.start();
t1.join();

t2.start();
t2.join();
```

This causes the main thread to wait for `t1` before even starting `t2`.

The tasks therefore do not get the same opportunity to execute concurrently.

### Better when tasks are independent

```java
t1.start();
t2.start();

t1.join();
t2.join();
```

This starts both tasks first and then waits for both.

---

## 18. Timed Join Example ⭐⭐⭐⭐⭐

```java
public class TimedJoinDemo {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        worker.join(1000);

        System.out.println("Worker alive: "
                + worker.isAlive());
    }
}
```

Because the worker sleeps for about 5 seconds while main waits only up to about 1 second, the worker will normally still be alive when the join returns.

---

# 19. Practice Code ⭐⭐⭐⭐⭐

Create:

`JoinPractice.java`

```java
public class JoinPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker started");

            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println("Worker finished");
        }, "worker");

        worker.start();

        System.out.println("Main waiting...");
        worker.join();

        System.out.println("Main continues after worker");
    }
}
```

Expected order:

```text
Worker started
Main waiting...
Worker finished
Main continues after worker
```

The exact relative position of `Main waiting...` and `Worker started` can vary, but the final line comes after the worker has terminated.

---

# 20. Practice Code — Multiple Workers ⭐⭐⭐⭐⭐

```java
public class MultipleJoinPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> work(2000), "worker-1");
        Thread second = new Thread(() -> work(1000), "worker-2");
        Thread third = new Thread(() -> work(1500), "worker-3");

        first.start();
        second.start();
        third.start();

        first.join();
        second.join();
        third.join();

        System.out.println("All workers completed");
    }

    private static void work(long millis) {
        try {
            System.out.println(
                    Thread.currentThread().getName()
                            + " started"
            );

            Thread.sleep(millis);

            System.out.println(
                    Thread.currentThread().getName()
                            + " finished"
            );
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Key observation

All three workers start before main waits for them.

The workers therefore execute concurrently.

---

# 21. Practice Code — Timed Join ⭐⭐⭐⭐⭐

```java
public class TimedJoinPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        long start = System.currentTimeMillis();

        worker.join(1000);

        long elapsed = System.currentTimeMillis() - start;

        System.out.println("Join waited approximately: "
                + elapsed + " ms");
        System.out.println("Worker alive: "
                + worker.isAlive());
    }
}
```

Expected conceptual result:

```text
Join waited approximately around 1000 ms
Worker alive: true
```

The exact elapsed time is not guaranteed.

---

# 22. Practice Code — Interrupting a Thread Waiting in `join()` ⭐⭐⭐⭐⭐

```java
public class JoinInterruptPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "worker");

        Thread waiter = new Thread(() -> {
            try {
                System.out.println("Waiter joining worker");
                worker.join();
                System.out.println("Waiter: worker completed");
            } catch (InterruptedException e) {
                System.out.println("Waiter interrupted");
                Thread.currentThread().interrupt();
            }
        }, "waiter");

        worker.start();
        waiter.start();

        Thread.sleep(1000);

        waiter.interrupt();

        worker.join();
        waiter.join();

        System.out.println("Main completed");
    }
}
```

### Important observation

`waiter.interrupt()` interrupts the thread waiting in `join()`.

It does **not** automatically interrupt `worker`.

---

# 23. Practice Exercise — Predict the Output ⭐⭐⭐⭐⭐

```java
Thread worker = new Thread(() -> {
    System.out.println("Worker");
});

worker.start();
worker.join();

System.out.println("Main");
```

### Question

Which statement must happen before `Main`?

### Answer

The worker must have terminated before `join()` returns, so the worker's task execution has completed before `Main` is printed.

---

# 24. Practice Exercise — Does `join()` Make Threads Sequential?

```java
t1.start();
t2.start();

t1.join();
t2.join();
```

### Answer

**No.**

`t1` and `t2` can execute concurrently because both were started before the joins.

The current thread waits for their completion.

---

# 25. Practice Exercise — Timed Join

```java
worker.start();
worker.join(1000);
System.out.println(worker.isAlive());
```

If `worker` needs 5 seconds to finish, can the output be:

```text
true
```

### Answer

Yes.

The timed join waits only up to approximately one second; it does not stop the worker.

---

# 26. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — `join()` stops the target

❌ False.

The target continues executing.

### Mistake 2 — `join()` interrupts the target

❌ False.

Only the current thread waits.

### Mistake 3 — `join()` makes all threads sequential

❌ False.

It only affects the caller's waiting behavior.

### Mistake 4 — `join(1000)` guarantees completion within 1 second

❌ False.

It only limits how long the caller waits.

### Mistake 5 — `join()` waits forever even if interrupted

❌ False.

It throws `InterruptedException` when the waiting thread is interrupted.

### Mistake 6 — Calling `join()` before `start()` starts the target

❌ No.

`join()` waits for the target's termination; it does not start the target thread.

---

# 27. Interview Questions

### Q1. What is `join()`?

`join()` makes the current thread wait until the target thread terminates.

### Q2. Which thread actually waits?

The thread that calls `join()`.

### Q3. Does `join()` stop the target thread?

No.

### Q4. What exception can `join()` throw?

`InterruptedException`.

### Q5. What is `join(timeout)`?

A bounded wait where the caller waits for at most the requested duration, unless the target terminates earlier or the wait is interrupted.

### Q6. Does `join()` make worker threads execute sequentially?

No. Starting workers first and then joining them allows them to execute concurrently.

### Q7. Difference between `sleep()` and `join()`?

`sleep()` waits for a duration; `join()` waits for another thread to terminate.

### Q8. Does interrupting a thread waiting in `join()` interrupt the target?

No. It interrupts the waiting/calling thread.

### Q9. Does `join()` release a monitor lock held by the caller?

`join()` itself is not a monitor-release mechanism like `wait()`. Do not confuse `join()` with `wait()`.

### Q10. Can you call `join()` without starting the target?

It does not start the target. Waiting behavior depends on the target's state; a thread that has not been started is not a running task to wait for.

---

# 28. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`Thread.join()` is used when the current thread needs to wait for another thread to complete. For example, after `worker.start()`, calling `worker.join()` causes the current thread, such as main, to wait until worker terminates. The important point is that `join()` does not stop, interrupt, or pause the target thread; the target continues executing normally while the caller waits. Java also provides timed versions such as `join(1000)`, where the caller waits for up to approximately one second and then continues if the target is still alive. `join()` is interruptible and throws `InterruptedException` if the waiting thread is interrupted. If multiple independent workers are needed, we normally start all of them first and then call `join()` on each so they can execute concurrently. In short, `sleep()` waits for time, while `join()` waits for thread completion."**

---

# 29. Quick Revision

```text
worker.start()
      ↓
worker executes
      ↓
main calls worker.join()
      ↓
main waits
      ↓
worker TERMINATED
      ↓
main continues
```

### Golden Rules ⭐⭐⭐⭐⭐

```text
join() → current thread waits
join() → waits for target thread
join() → does NOT stop target
join(timeout) → bounded wait
join() → interruptible
```

### Memory Trick

> **`sleep()` = wait for time | `join()` = wait for thread**

---

# 30. Completion Checklist

- [x] `join()` fundamentals
- [x] Current thread vs target thread
- [x] Thread lifecycle relationship
- [x] `InterruptedException`
- [x] Interrupting a waiting thread
- [x] `join(long)`
- [x] `join(long, int)`
- [x] Multiple worker threads
- [x] Concurrent workers + joins
- [x] `join()` vs `sleep()`
- [x] `join()` vs `wait()`
- [x] Timed join practice
- [x] Multiple-thread practice
- [x] Interrupt practice
- [x] Output prediction exercises
- [x] Interview traps
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.8 — `sleep()`](../08-sleep/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.10 — `yield()`**