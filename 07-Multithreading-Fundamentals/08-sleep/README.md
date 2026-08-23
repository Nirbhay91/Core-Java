# 7.8 — `sleep()`

## 🎯 Objective

Understand how `Thread.sleep()` pauses the **currently executing thread** for a specified amount of time, how it affects thread state, interruption, locks, and common interview scenarios.

---

## 1. What is `Thread.sleep()`? ⭐⭐⭐⭐⭐

`Thread.sleep()` temporarily pauses the thread that is currently executing the call.

```java
Thread.sleep(1000);
```

means:

```text
Pause current thread for approximately 1000 ms
```

It does **not** pause the entire JVM or all application threads.

---

## 2. Basic Example

```java
public class SleepDemo {

    public static void main(String[] args)
            throws InterruptedException {

        System.out.println("Before sleep");

        Thread.sleep(2000);

        System.out.println("After sleep");
    }
}
```

The main thread pauses for approximately 2 seconds before continuing.

---

## 3. Why is `sleep()` Static? ⭐⭐⭐⭐

The API is:

```java
Thread.sleep(...)
```

because sleep applies to the **currently executing thread**, not to an arbitrary `Thread` object.

For example:

```java
Thread worker = new Thread(...);
worker.start();
```

Inside the worker:

```java
Thread.sleep(1000);
```

pauses the worker because the worker is the current executing thread at that point.

### Important interview point

Calling:

```java
worker.sleep(1000);
```

is misleading. Since `sleep()` is static, it still affects the thread executing that statement, not necessarily `worker`.

Prefer:

```java
Thread.sleep(1000);
```

---

## 4. `sleep()` Does Not Create a New Thread

```java
Thread.sleep(1000);
```

does not create another thread.

It only changes the current thread's execution state temporarily.

Conceptually:

```text
RUNNABLE
   |
   | sleep()
   ↓
TIMED_WAITING
   |
   | timeout / interruption
   ↓
RUNNABLE
```

At the Java API level, sleeping causes the thread to enter `TIMED_WAITING`.

---

## 5. `sleep(long millis)`

The simplest overload is:

```java
Thread.sleep(1000);
```

where `1000` means approximately 1000 milliseconds.

```text
1000 ms = 1 second
2000 ms = 2 seconds
500 ms  = 0.5 seconds
```

The actual elapsed time can be longer because of scheduling, OS timing, contention, and other factors.

---

## 6. `sleep(long millis, int nanos)`

Java also provides:

```java
Thread.sleep(long millis, int nanos)
```

The nanosecond component provides additional sub-millisecond precision within the API's constraints.

Example:

```java
Thread.sleep(100, 500_000);
```

This requests approximately:

```text
100 ms + 500,000 ns
```

Do not treat this as a guarantee of exact wake-up timing.

---

## 7. `sleep()` Throws `InterruptedException` ⭐⭐⭐⭐⭐

The method is interruptible.

Therefore code normally needs to handle:

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

or propagate it:

```java
public static void main(String[] args)
        throws InterruptedException {

    Thread.sleep(1000);
}
```

---

## 8. What Happens When a Sleeping Thread Is Interrupted? ⭐⭐⭐⭐⭐

Suppose:

```java
Thread worker = new Thread(() -> {
    try {
        Thread.sleep(10_000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        System.out.println("Interrupted");
    }
});
```

Another thread calls:

```java
worker.interrupt();
```

The sleeping thread is woken by interruption and `sleep()` throws `InterruptedException`.

### Important

`interrupt()` does **not** forcibly kill the thread.

It provides a cooperative interruption mechanism.

---

## 9. Why Restore the Interrupt Flag? ⭐⭐⭐⭐⭐

When `InterruptedException` is caught, the interrupt status has been cleared as part of the exception mechanism.

A common best practice is:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

This restores the interrupt status so higher-level code can observe that interruption was requested.

In application code, you may alternatively propagate the exception when the surrounding API allows it.

---

## 10. Does `sleep()` Release a Lock? ⭐⭐⭐⭐⭐

**No.**

This is one of the most important interview questions.

If a thread holds an intrinsic monitor lock and calls `sleep()`, it remains the owner of that monitor while sleeping.

Example:

```java
synchronized (lock) {
    Thread.sleep(5000);
}
```

The thread continues to hold `lock` during the sleep.

Compare this with `wait()`:

```text
sleep() → pauses thread, keeps monitor lock
wait()  → waits and releases monitor lock
```

---

## 11. `sleep()` vs `wait()` ⭐⭐⭐⭐⭐

| `sleep()` | `wait()` |
|---|---|
| Method of `Thread` | Method of `Object` |
| Static API | Instance method |
| Pauses current thread | Waits for notification/timeout |
| Does not release monitor lock | Releases monitor lock when waiting |
| Can be called without owning a monitor | Must be called while owning the object's monitor |
| Commonly used for timing/delay | Used for thread coordination |
| Throws `InterruptedException` | Throws `InterruptedException` |

---

## 12. `sleep()` vs `yield()`

| `sleep()` | `yield()` |
|---|---|
| Requests a timed pause | Scheduler hint |
| Has a specified duration | No duration |
| Causes `TIMED_WAITING` | Remains `RUNNABLE` at Java API level |
| Can throw `InterruptedException` | Does not throw `InterruptedException` |
| More explicit delay | No guarantee that another thread runs |

---

## 13. Does `sleep()` Guarantee Exact Timing?

No.

If you write:

```java
long start = System.currentTimeMillis();
Thread.sleep(1000);
long end = System.currentTimeMillis();
```

You should not assume:

```text
end - start == exactly 1000 ms
```

The sleep duration is a minimum-ish scheduling request rather than a guarantee of exact wake-up time.

The thread may resume later depending on scheduling and system conditions.

---

## 14. `sleep(0)`

```java
Thread.sleep(0);
```

requests a zero-duration sleep. It does not provide a useful guaranteed scheduling behavior and should not be confused with `yield()`.

Do not rely on it for synchronization or ordering.

---

## 15. Thread State During Sleep ⭐⭐⭐⭐⭐

Example:

```java
Thread worker = new Thread(() -> {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

worker.start();
```

While sleeping, its Java-level state is:

```java
Thread.State.TIMED_WAITING
```

You can observe it with:

```java
worker.getState();
```

The exact observation depends on timing.

---

## 16. `sleep()` Inside a Loop

A common use is periodic work:

```java
while (!Thread.currentThread().isInterrupted()) {
    process();

    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        break;
    }
}
```

This pattern is useful because interruption can stop the loop cleanly.

---

# 17. Practice Code ⭐⭐⭐⭐⭐

Create:

`SleepPractice.java`

```java
public class SleepPractice {

    public static void main(String[] args)
            throws InterruptedException {

        System.out.println("Start: "
                + System.currentTimeMillis());

        Thread.sleep(2000);

        System.out.println("End:   "
                + System.currentTimeMillis());
    }
}
```

### Observe

The elapsed time should be around 2 seconds or slightly more, not necessarily exactly 2 seconds.

---

# 18. Practice Code — Multiple Threads ⭐⭐⭐⭐⭐

```java
public class MultipleThreadSleepPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> work("first"), "worker-1");
        Thread second = new Thread(() -> work("second"), "worker-2");

        first.start();
        second.start();

        first.join();
        second.join();
    }

    private static void work(String name) {
        try {
            for (int i = 1; i <= 3; i++) {
                System.out.println(
                        Thread.currentThread().getName()
                                + " - " + name
                                + " - step " + i
                );

                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Observe

Both threads can make progress independently while each sleeps.

This demonstrates:

```text
worker-1 sleeps
worker-2 can execute
worker-2 sleeps
worker-1 can execute
```

---

# 19. Practice Code — Interrupted Sleep ⭐⭐⭐⭐⭐

```java
public class InterruptedSleepPractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                System.out.println("Worker sleeping...");
                Thread.sleep(10_000);
                System.out.println("Worker woke normally");
            } catch (InterruptedException e) {
                System.out.println("Worker interrupted");
                Thread.currentThread().interrupt();
            }
        }, "worker");

        worker.start();

        Thread.sleep(1000);

        System.out.println("Main interrupts worker");
        worker.interrupt();

        worker.join();

        System.out.println("Worker finished");
    }
}
```

Expected conceptual flow:

```text
Worker sleeping...
Main interrupts worker
Worker interrupted
Worker finished
```

---

# 20. Practice Code — Sleep Does Not Release Lock ⭐⭐⭐⭐⭐

```java
public class SleepLockPractice {

    private static final Object LOCK = new Object();

    public static void main(String[] args)
            throws InterruptedException {

        Thread first = new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("First acquired lock");

                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }

                System.out.println("First releasing lock");
            }
        }, "first");

        Thread second = new Thread(() -> {
            System.out.println("Second trying to acquire lock");

            synchronized (LOCK) {
                System.out.println("Second acquired lock");
            }
        }, "second");

        first.start();
        Thread.sleep(500);
        second.start();

        first.join();
        second.join();
    }
}
```

### Key observation

Even though `first` is sleeping, it still owns `LOCK`.

Therefore `second` cannot enter the synchronized block until `first` exits it.

---

# 21. Practice Exercise — Predict the State ⭐⭐⭐⭐⭐

```java
Thread worker = new Thread(() -> {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

worker.start();
```

While the worker is sleeping, what should:

```java
worker.getState()
```

normally return?

### Answer

```text
TIMED_WAITING
```

The observation is timing-dependent if checked too early or after the sleep ends.

---

# 22. Practice Exercise — Does Sleep Release Lock?

Consider:

```java
synchronized (lock) {
    Thread.sleep(5000);
}
```

### Question

Can another thread immediately acquire `lock` while the first thread is sleeping?

### Answer

**No.**

The sleeping thread continues to hold the intrinsic monitor until it leaves the synchronized block.

---

# 23. Practice Exercise — Which Thread Sleeps?

```java
Thread worker = new Thread(() -> {
    System.out.println("Worker");
}, "worker");

worker.start();
Thread.sleep(2000);
```

### Question

Which thread sleeps?

### Answer

The **main/current thread**, because `Thread.sleep()` affects the thread executing that call.

It does not tell `worker` to sleep.

---

# 24. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — `sleep()` sleeps the Thread object

❌ Not exactly.

It pauses the **currently executing thread**.

### Mistake 2 — `sleep()` releases the lock

❌ False.

`sleep()` does not release an intrinsic monitor lock.

### Mistake 3 — Sleep duration is exact

❌ False.

The thread may resume later than requested.

### Mistake 4 — `sleep()` stops other threads

❌ False.

Only the current thread is affected.

### Mistake 5 — `interrupt()` kills a sleeping thread

❌ False.

It causes `sleep()` to throw `InterruptedException`; the thread should handle the interruption cooperatively.

### Mistake 6 — Catch and ignore interruption

Avoid:

```java
catch (InterruptedException e) {
}
```

unless there is a deliberate reason and the interruption semantics are handled elsewhere.

A common response is:

```java
Thread.currentThread().interrupt();
```

---

# 25. Interview Questions

### Q1. What does `Thread.sleep()` do?

It pauses the currently executing thread for the requested duration.

### Q2. Is `sleep()` static?

Yes.

```java
Thread.sleep(...)
```

### Q3. Does `sleep()` release a lock?

No.

### Q4. What state does a sleeping thread enter?

`TIMED_WAITING`.

### Q5. Can another thread interrupt a sleeping thread?

Yes, using `interrupt()`.

### Q6. What exception does `sleep()` throw?

`InterruptedException`.

### Q7. Does `sleep()` guarantee exact timing?

No.

### Q8. Does sleeping one thread pause all threads?

No.

### Q9. Difference between `sleep()` and `wait()`?

`sleep()` does not release a monitor lock and is used for timed pausing; `wait()` releases the object's monitor while waiting and is used for coordination.

### Q10. Why restore interrupt status after catching `InterruptedException`?

Because interruption is cooperative, and restoring the status allows higher-level code to observe the interruption request.

---

# 26. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`Thread.sleep()` is a static method that pauses the currently executing thread for a specified duration. During sleep, the Java thread enters `TIMED_WAITING`. Sleep does not create a new thread and does not pause other threads. An important interview point is that `sleep()` does not release an intrinsic monitor lock, so if a thread sleeps inside a synchronized block it continues holding that lock. `sleep()` is interruptible and throws `InterruptedException` when another thread interrupts the sleeping thread. In a catch block, we commonly restore the interrupt status using `Thread.currentThread().interrupt()`. Also, sleep is not an exact timing guarantee; the thread may resume later depending on scheduling and system conditions. The key distinction is: sleep pauses the current thread, but it does not release its monitor lock."**

---

# 27. Quick Revision

```text
Thread.sleep(time)
       ↓
Current thread pauses
       ↓
TIMED_WAITING
       ↓
timeout / interrupt
       ↓
continues or handles interruption
```

### Golden Rules ⭐⭐⭐⭐⭐

```text
sleep() → current thread
sleep() → TIMED_WAITING
sleep() → does NOT release lock
sleep() → interruptible
sleep() → timing is not exact
```

### Memory Trick

> **Sleep pauses the thread, not the lock.**

---

# 28. Completion Checklist

- [x] `Thread.sleep()` fundamentals
- [x] Static nature of `sleep()`
- [x] Milliseconds and nanoseconds overload
- [x] `TIMED_WAITING`
- [x] Interrupted sleep
- [x] Interrupt flag restoration
- [x] Lock behavior
- [x] `sleep()` vs `wait()`
- [x] `sleep()` vs `yield()`
- [x] Timing accuracy
- [x] Loop usage
- [x] Practice code
- [x] Lock practice
- [x] Interruption practice
- [x] Output/state exercises
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.7 — `start()` vs `run()`](../07-start-vs-run/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.9 — `join()`**