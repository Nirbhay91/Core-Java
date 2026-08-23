# 7.10 — `Thread.yield()`

## 🎯 Objective

Understand `Thread.yield()` as a **scheduler hint**, what it does and does not guarantee, its Java thread state behavior, lock behavior, and common interview traps.

---

## 1. What is `yield()`? ⭐⭐⭐⭐⭐

```java
Thread.yield();
```

`yield()` is a static method that gives the scheduler a **hint** that the current thread is willing to give other runnable threads an opportunity to execute.

Important:

> `yield()` is only a **hint**. The JVM/OS scheduler is not required to honor it.

So you cannot use `yield()` to guarantee thread ordering or synchronization.

---

## 2. Basic Example

```java
public class YieldDemo {

    public static void main(String[] args) {

        Thread worker = new Thread(() -> {
            for (int i = 1; i <= 5; i++) {
                System.out.println(
                        Thread.currentThread().getName()
                                + " - " + i
                );

                Thread.yield();
            }
        }, "worker");

        worker.start();
    }
}
```

The call may give another runnable thread an opportunity to execute, but there is no guaranteed output ordering.

---

## 3. Why is `yield()` Static? ⭐⭐⭐⭐

The API is:

```java
Thread.yield();
```

because the hint applies to the **currently executing thread**.

It does not mean:

```java
someOtherThread.yield();
```

will force that other thread to yield.

The clean mental model is:

```text
CURRENT THREAD
      |
      | yield()
      ↓
"Scheduler, I am willing to give others a chance."
```

---

## 4. `yield()` Is Only a Hint ⭐⭐⭐⭐⭐

This is the most important interview point.

```java
Thread.yield();
```

does **not** guarantee:

- another thread will run
- the current thread will stop immediately
- the current thread will remain paused for any duration
- a specific thread will be selected next
- fair scheduling
- context switching

The scheduler may completely ignore the hint.

---

## 5. Thread State After `yield()` ⭐⭐⭐⭐⭐

A common interview trap is saying:

```text
RUNNING → READY
```

as if Java defines those states.

Java's `Thread.State` enum does **not** have `RUNNING` or `READY` states.

The Java-level states include:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

A thread calling `yield()` remains in the Java-level **`RUNNABLE`** state.

The distinction between running and ready-to-run is an OS/scheduler implementation detail, not a separate portable Java `Thread.State`.

---

## 6. `yield()` vs `sleep()` ⭐⭐⭐⭐⭐

| `yield()` | `sleep()` |
|---|---|
| Scheduler hint | Timed pause request |
| No specified duration | Has requested duration |
| Remains `RUNNABLE` at Java API level | Enters `TIMED_WAITING` |
| No `InterruptedException` | Throws `InterruptedException` |
| Scheduler may ignore it | Current thread pauses for requested interval, subject to scheduling |
| No timing guarantee | No exact wake-up guarantee |

### Memory trick

```text
yield() → "Others may go first"
sleep() → "Pause me for a while"
```

---

## 7. `yield()` vs `join()`

| `yield()` | `join()` |
|---|---|
| Scheduler hint | Waits for target thread |
| Current thread remains `RUNNABLE` | Calling thread waits |
| No completion guarantee | Waits until target terminates or timeout/interruption |
| Does not coordinate completion | Used for thread completion coordination |

---

## 8. `yield()` vs `wait()` ⭐⭐⭐⭐⭐

| `yield()` | `wait()` |
|---|---|
| Method of `Thread` | Method of `Object` |
| Scheduler hint | Thread coordination mechanism |
| Remains `RUNNABLE` | Enters `WAITING`/`TIMED_WAITING` depending on overload |
| Does not release monitor lock | Releases the object's monitor while waiting |
| No notification required | Usually resumes through notification, interruption, or timeout |

---

## 9. Does `yield()` Release a Lock? ⭐⭐⭐⭐⭐

**No.**

If a thread owns an intrinsic monitor:

```java
synchronized (lock) {
    Thread.yield();
}
```

`yield()` does not release `lock`.

This is another major interview distinction:

```text
yield() → does NOT release monitor
sleep() → does NOT release monitor
wait()  → releases monitor while waiting
```

---

## 10. Does `yield()` Guarantee Another Thread Will Run?

**No.**

Example:

```java
while (true) {
    Thread.yield();
}
```

You cannot conclude that another thread will definitely get CPU time because of the yield calls.

The scheduler decides what happens.

---

## 11. Does `yield()` Guarantee Fairness?

No.

It is not a fairness mechanism and should not be used to implement:

- locks
- ordering
- thread communication
- producer-consumer coordination
- rate limiting
- deterministic scheduling

Use proper concurrency constructs instead.

---

## 12. Is `yield()` Useful in Production Code?

It can have specialized low-level uses, but application code generally should **not depend on it for correctness**.

If correctness depends on another thread getting a chance to run, `yield()` is the wrong mechanism.

Use appropriate primitives such as:

- `join()`
- `wait()/notify()` where appropriate
- `Lock` / `Condition`
- `CountDownLatch`
- `Semaphore`
- `ExecutorService`
- other concurrency utilities

---

# 13. Practice Code ⭐⭐⭐⭐⭐

Create:

`YieldPractice.java`

```java
public class YieldPractice {

    public static void main(String[] args) {

        Thread first = new Thread(() -> work(), "worker-1");
        Thread second = new Thread(() -> work(), "worker-2");

        first.start();
        second.start();
    }

    private static void work() {
        for (int i = 1; i <= 10; i++) {
            System.out.println(
                    Thread.currentThread().getName()
                            + " -> " + i
            );

            Thread.yield();
        }
    }
}
```

### Observe

Run the program multiple times.

Do **not** expect a deterministic pattern such as:

```text
worker-1
worker-2
worker-1
worker-2
```

The scheduler may produce different results.

---

# 14. Practice Code — `yield()` Does Not Guarantee Ordering ⭐⭐⭐⭐⭐

```java
public class YieldOrderingPractice {

    public static void main(String[] args) {

        Thread first = new Thread(() -> {
            System.out.println("First - before yield");
            Thread.yield();
            System.out.println("First - after yield");
        });

        Thread second = new Thread(() -> {
            System.out.println("Second - before yield");
            Thread.yield();
            System.out.println("Second - after yield");
        });

        first.start();
        second.start();
    }
}
```

### Interview question

Can you guarantee:

```text
First - before yield
Second - before yield
Second - after yield
First - after yield
```

### Answer

**No.**

`yield()` provides no such ordering guarantee.

---

# 15. Practice Code — Check `Thread.State` ⭐⭐⭐⭐⭐

```java
public class YieldStatePractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            for (int i = 0; i < 1_000_000; i++) {
                Thread.yield();
            }
        }, "worker");

        worker.start();

        Thread.sleep(10);

        System.out.println("State: " + worker.getState());

        worker.join();
    }
}
```

The observed state is timing-dependent, but while executing/yielding it is represented at the Java API level as `RUNNABLE`, not a separate `READY` state.

---

# 16. Practice Code — `yield()` Inside Synchronized Block ⭐⭐⭐⭐⭐

```java
public class YieldLockPractice {

    private static final Object LOCK = new Object();

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("First acquired lock");
                Thread.yield();
                System.out.println("First still owns lock");
            }
        }, "first");

        Thread second = new Thread(() -> {
            System.out.println("Second trying for lock");

            synchronized (LOCK) {
                System.out.println("Second acquired lock");
            }
        }, "second");

        first.start();
        Thread.sleep(100);
        second.start();

        first.join();
        second.join();
    }
}
```

### Key point

`yield()` does not release `LOCK`.

The second thread can acquire the lock only after the first thread exits the synchronized block.

---

# 17. Practice Exercise — `yield()` Guarantee ⭐⭐⭐⭐⭐

```java
Thread.yield();
```

### Question

Does this guarantee that another thread will execute immediately?

### Answer

**No.**

It is only a scheduler hint.

---

# 18. Practice Exercise — Thread State

### Question

Does Java define a `READY` state in `Thread.State`?

### Answer

**No.**

Java defines `RUNNABLE`, which covers the Java-level runnable state. OS-level distinctions such as running vs ready are implementation details.

---

# 19. Practice Exercise — Lock

```java
synchronized (lock) {
    Thread.yield();
}
```

### Question

Does `yield()` release `lock`?

### Answer

**No.**

The thread continues to own the intrinsic monitor while it remains inside the synchronized block.

---

# 20. Practice Exercise — `yield()` vs `sleep()`

### Question

Which one enters `TIMED_WAITING`?

### Answer

```java
Thread.sleep(...);
```

A thread calling `yield()` remains `RUNNABLE` at the Java API level.

---

# 21. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — `yield()` guarantees another thread runs

❌ False.

It is only a scheduler hint.

### Mistake 2 — `yield()` puts the thread into `WAITING`

❌ False.

At the Java API level it remains `RUNNABLE`.

### Mistake 3 — Java has `RUNNING` and `READY` states

❌ False.

Those are not separate values in `Thread.State`.

### Mistake 4 — `yield()` releases a lock

❌ False.

It does not release an intrinsic monitor.

### Mistake 5 — `yield()` guarantees fairness

❌ False.

No fairness guarantee exists.

### Mistake 6 — `yield()` should be used for synchronization

❌ False.

Use proper synchronization/concurrency primitives.

---

# 22. Interview Questions

### Q1. What is `Thread.yield()`?

A static scheduler hint indicating that the current thread is willing to give other runnable threads an opportunity to execute.

### Q2. Is `yield()` guaranteed to work?

No. The scheduler may ignore the hint.

### Q3. What is the Java thread state after `yield()`?

The thread remains `RUNNABLE` at the Java API level.

### Q4. Does `yield()` release a monitor lock?

No.

### Q5. Does `yield()` throw `InterruptedException`?

No.

### Q6. Difference between `yield()` and `sleep()`?

`yield()` is a scheduler hint with no duration; `sleep()` requests a timed pause and enters `TIMED_WAITING`.

### Q7. Can `yield()` guarantee thread ordering?

No.

### Q8. Can `yield()` be used to solve race conditions?

No. It does not provide synchronization or atomicity.

### Q9. Does `yield()` force a context switch?

No. It is a hint and does not guarantee a context switch.

### Q10. Is `yield()` a synchronization mechanism?

No.

---

# 23. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`Thread.yield()` is a static method that provides a hint to the scheduler that the currently executing thread is willing to give other runnable threads an opportunity to execute. The important point is that it is only a hint; the scheduler can ignore it, so it cannot guarantee another thread will run, a context switch will occur, or any ordering/fairness will happen. At the Java API level, a thread calling `yield()` remains in the `RUNNABLE` state; Java does not define separate RUNNING and READY states in `Thread.State`. Also, `yield()` does not release an intrinsic monitor lock. Unlike `sleep()`, it has no specified waiting duration and does not throw `InterruptedException`. Therefore, `yield()` should not be used for synchronization or correctness; proper concurrency primitives should be used instead."**

---

# 24. Quick Revision

```text
yield()
   ↓
Current thread gives scheduler a hint
   ↓
"Other runnable threads may get a chance"
   ↓
Scheduler may honor OR ignore
```

### Golden Rules ⭐⭐⭐⭐⭐

```text
yield() → scheduler hint
yield() → no guarantee
yield() → RUNNABLE at Java level
yield() → does NOT release lock
yield() → no InterruptedException
yield() → NOT a synchronization mechanism
```

### Memory Trick

> **Yield = Hint, not Guarantee.**

---

# 25. Completion Checklist

- [x] `Thread.yield()` fundamentals
- [x] Static/current-thread behavior
- [x] Scheduler hint
- [x] No scheduling guarantee
- [x] Java `Thread.State` behavior
- [x] RUNNABLE nuance
- [x] `yield()` vs `sleep()`
- [x] `yield()` vs `join()`
- [x] `yield()` vs `wait()`
- [x] Lock behavior
- [x] Practice code
- [x] State practice
- [x] Lock practice
- [x] Ordering exercises
- [x] Interview traps
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.9 — `join()`](../09-join/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.11 — Race Condition**