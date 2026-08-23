# 7.5 — `Thread.State` and `RUNNABLE`

## 🎯 Objective

Understand Java's `Thread.State` enum in depth, especially the meaning of `RUNNABLE`, how it differs from OS-level **ready/running** terminology, and how to observe thread states safely for debugging and interview discussions.

---

## 1. What is `Thread.State`?

Java exposes the lifecycle state of a thread through the nested enum:

```java
Thread.State
```

Java defines exactly **six** states:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

You can inspect the current Java-level state with:

```java
thread.getState();
```

Example:

```java
Thread thread = new Thread(() -> {
    System.out.println("Running...");
});

System.out.println(thread.getState());
```

Before `start()` the result is:

```text
NEW
```

---

# 2. Why is `RUNNABLE` Important? ⭐⭐⭐⭐⭐

`RUNNABLE` is often misunderstood because developers expect Java to expose separate states such as:

```text
READY
RUNNING
```

Java does **not** define those as separate `Thread.State` values.

Instead, Java's `RUNNABLE` state represents a thread that is eligible to run and may be actively executing or waiting for CPU availability.

### Interview-safe statement

> Java does not expose separate `READY` and `RUNNING` states in `Thread.State`; both situations are represented by `RUNNABLE` at the Java API level.

---

# 3. `Thread.State` Enum

The enum is conceptually:

```java
public enum Thread.State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED
}
```

These states describe the Java-level lifecycle of a thread.

They should not be treated as a complete representation of every OS scheduler state.

---

# 4. NEW → RUNNABLE

Creating a thread:

```java
Thread t = new Thread(() -> {
    System.out.println("Task running");
});
```

At this point:

```text
NEW
```

After:

```java
t.start();
```

the thread becomes eligible for execution and enters the Java-level `RUNNABLE` state.

```text
NEW
  ↓ start()
RUNNABLE
```

---

# 5. Does `start()` Mean the Thread Immediately Runs?

No.

```java
t.start();
```

means the JVM can schedule the thread for execution. It does not mean that the thread immediately receives CPU time.

Conceptually:

```text
start()
  ↓
RUNNABLE
  ↓
Scheduler decides when it executes
```

Therefore, this is wrong:

> `start()` guarantees that the new thread executes before the next line in the main thread.

There is no such ordering guarantee.

---

# 6. RUNNABLE Does Not Guarantee CPU Execution

Suppose:

```java
Thread t = new Thread(() -> {
    System.out.println("Worker");
});

t.start();
System.out.println("Main");
```

Possible output:

```text
Main
Worker
```

or:

```text
Worker
Main
```

The scheduler determines the actual execution order.

### Remember

```text
RUNNABLE ≠ guaranteed CPU execution right now
```

It means the thread is eligible to run at the Java level.

---

# 7. RUNNABLE vs RUNNING ⭐⭐⭐⭐⭐

### Java API

Java exposes:

```text
RUNNABLE
```

It does not expose:

```text
RUNNING
```

as a separate `Thread.State`.

### OS-level discussion

An operating system scheduler may internally distinguish states such as:

```text
READY
RUNNING
```

But these distinctions are not represented as separate Java `Thread.State` enum constants.

### Interview answer

> In Java, `RUNNABLE` covers the Java-level state for a thread that is ready/eligible to execute as well as one that is actually executing. The OS may maintain finer-grained scheduling states internally.

---

# 8. RUNNABLE vs BLOCKED

### RUNNABLE

The thread is eligible to execute.

### BLOCKED

The thread is waiting to acquire an intrinsic monitor lock.

Example:

```java
synchronized (lock) {
    // critical section
}
```

If another thread already owns `lock`:

```text
Thread A → owns lock
Thread B → requests lock
Thread B → BLOCKED
```

Once the lock becomes available and the thread can proceed, it can become `RUNNABLE` again.

---

# 9. RUNNABLE vs WAITING

### RUNNABLE

```text
Eligible to execute
```

### WAITING

The thread waits indefinitely for another action.

Examples:

```java
object.wait();
thread.join();
LockSupport.park();
```

Conceptually:

```text
RUNNABLE
   ↓ wait()
WAITING
   ↓ notification / completion / unpark
RUNNABLE
```

---

# 10. RUNNABLE vs TIMED_WAITING

### RUNNABLE

Eligible to execute.

### TIMED_WAITING

Waiting for a bounded amount of time.

Examples:

```java
Thread.sleep(1000);
object.wait(1000);
thread.join(1000);
```

Conceptually:

```text
RUNNABLE
   ↓ sleep(1000)
TIMED_WAITING
   ↓ timeout / interrupt
RUNNABLE
```

---

# 11. RUNNABLE → TERMINATED

When the thread's `run()` method completes normally:

```text
RUNNABLE
   ↓
run() completes
   ↓
TERMINATED
```

Example:

```java
Thread t = new Thread(() -> {
    System.out.println("Task");
});

t.start();
```

After the task completes:

```java
t.getState();
```

will eventually return:

```text
TERMINATED
```

---

# 12. State Observation is a Snapshot ⭐⭐⭐⭐⭐

```java
Thread.State state = t.getState();
```

returns the thread's state at that moment.

The state can change immediately after the call.

Therefore:

```java
if (t.getState() == Thread.State.RUNNABLE) {
    // do something
}
```

should not generally be used as a synchronization mechanism.

### Correct mindset

```text
getState()
   ↓
Diagnostic / monitoring information
```

not:

```text
getState()
   ↓
Synchronization guarantee
```

---

# 13. Why State Observation Can Be Nondeterministic

Consider:

```java
Thread t = new Thread(() -> {
    System.out.println("Running");
});

t.start();
System.out.println(t.getState());
```

You may observe different states depending on timing.

For a very short task, it is even possible for the thread to have already reached:

```text
TERMINATED
```

before `getState()` is executed.

Therefore, do not write interview answers that assume:

```text
start() → guaranteed observation of RUNNABLE
```

at an arbitrary later line.

---

# 14. Practical State Demo

```java
public class ThreadStateDemo {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "worker");

        System.out.println("Before start: " + worker.getState());

        worker.start();

        System.out.println("After start: " + worker.getState());

        Thread.sleep(200);

        System.out.println("During sleep: " + worker.getState());

        worker.join();

        System.out.println("After completion: " + worker.getState());
    }
}
```

Typical output:

```text
Before start: NEW
After start: RUNNABLE
During sleep: TIMED_WAITING
After completion: TERMINATED
```

The exact state observed immediately after `start()` is timing-dependent, so this output should be understood as a typical observation, not a guaranteed sequence at every print statement.

---

# 15. Practice Code ⭐⭐⭐⭐⭐

Create the following practice class in the same topic folder:

```java
public class ThreadStatePractice {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(1500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "worker-thread");

        // 1. NEW
        System.out.println("1. " + worker.getState());

        // 2. Start the thread
        worker.start();

        // 3. Observe state after start
        System.out.println("2. " + worker.getState());

        // 4. Give worker time to enter sleep()
        Thread.sleep(100);

        // 5. TIMED_WAITING is expected here
        System.out.println("3. " + worker.getState());

        // 6. Wait for worker to finish
        worker.join();

        // 7. TERMINATED
        System.out.println("4. " + worker.getState());
    }
}
```

### Expected conceptual sequence

```text
NEW
  ↓ start()
RUNNABLE
  ↓ sleep()
TIMED_WAITING
  ↓ timeout
RUNNABLE
  ↓ run() completes
TERMINATED
```

Again, `getState()` is timing-sensitive, so exact intermediate observations can vary.

---

# 16. Practice Exercise 1 — Detect RUNNABLE

Create a worker that performs a CPU-intensive loop:

```java
Thread worker = new Thread(() -> {
    long sum = 0;

    for (long i = 0; i < 1_000_000_000L; i++) {
        sum += i;
    }

    System.out.println(sum);
});
```

Start it and repeatedly inspect:

```java
System.out.println(worker.getState());
```

### Goal

Observe that while the task is executing, the Java-level state can be `RUNNABLE`.

Do not interpret `RUNNABLE` as a guarantee that the CPU is executing the thread at the exact instant of observation.

---

# 17. Practice Exercise 2 — Compare All Common States

Build three threads:

```text
Thread A → CPU work → RUNNABLE
Thread B → waits for monitor → BLOCKED
Thread C → sleep() → TIMED_WAITING
```

Then use:

```java
getState()
```

to inspect them.

This exercise prepares you for the next topics:

- `synchronized`
- `sleep()`
- `wait()`
- thread coordination

---

# 18. Practice Exercise 3 — State Transition Logger

Create a monitoring loop:

```java
while (worker.isAlive()) {
    System.out.println(worker.getState());
    Thread.sleep(100);
}
```

Observe how the state changes over time.

### Important

This is for learning/diagnostics only. Do not use polling `getState()` as a synchronization strategy.

---

# 19. Important Interview Traps ⭐⭐⭐⭐⭐

### Trap 1 — Java has RUNNING state

❌ No.

There are six official Java states and `RUNNABLE` is the relevant state.

### Trap 2 — RUNNABLE means definitely executing on CPU

❌ No.

It means eligible to run at the Java level; scheduler/OS details determine actual CPU execution.

### Trap 3 — `start()` guarantees immediate execution

❌ No.

It makes the thread eligible for scheduling.

### Trap 4 — `getState()` gives a permanent state

❌ No.

It returns a snapshot.

### Trap 5 — `RUNNABLE` and `BLOCKED` are the same

❌ No.

`BLOCKED` specifically means waiting to acquire an intrinsic monitor lock.

### Trap 6 — `RUNNABLE` is the only state during execution

At the Java API level, active execution is represented by `RUNNABLE`, but a thread may transition to other states during its lifetime based on synchronization/waiting operations.

---

# 20. Interview Questions

### Q1. What is `Thread.State`?

It is the Java enum representing the lifecycle state of a thread.

### Q2. How many states does Java define?

Six: `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, and `TERMINATED`.

### Q3. Does Java have a `RUNNING` state?

No. Java's `Thread.State` does not define a separate `RUNNING` state.

### Q4. What does `RUNNABLE` mean?

The thread is eligible to run and may be actively executing or waiting for CPU availability; Java does not expose those as separate states.

### Q5. Does `start()` guarantee immediate execution?

No. It makes the thread eligible for scheduling.

### Q6. What does `getState()` return?

The current Java-level state as a snapshot at the time of the call.

### Q7. Can a thread move from RUNNABLE to BLOCKED?

Yes, if it attempts to acquire an intrinsic monitor lock held by another thread.

### Q8. Can RUNNABLE become WAITING?

Yes, through operations such as `wait()`, an untimed `join()`, or `park()`.

### Q9. Why can `getState()` produce different results on different runs?

Because thread scheduling and timing are nondeterministic.

### Q10. Is `getState()` a synchronization mechanism?

No. It is mainly useful for diagnostics, monitoring, and debugging.

---

# 21. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"`Thread.State` is Java's enum for representing the lifecycle state of a thread. It has six states: NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, and TERMINATED. RUNNABLE is especially important because Java does not expose separate READY and RUNNING states. A RUNNABLE thread is eligible to execute and may actually be executing or waiting for CPU availability. Calling `start()` makes a new thread eligible for scheduling, but it does not guarantee immediate execution. `getState()` returns a snapshot, so the value can change immediately afterward and should not be used for synchronization. BLOCKED means waiting for an intrinsic monitor lock, while WAITING and TIMED_WAITING represent indefinite and bounded waiting respectively."**

---

# 22. Quick Revision

```text
Thread.State = 6 states

NEW
 ↓ start()
RUNNABLE
 ↓
 ├── BLOCKED          → monitor lock
 ├── WAITING          → indefinite wait
 └── TIMED_WAITING    → timed wait
 ↓
RUNNABLE
 ↓
TERMINATED
```

### Most Important

```text
No separate RUNNING state in Java.

RUNNABLE = Java-level runnable/eligible state
```

### `getState()`

```text
Snapshot only
↓
Useful for diagnostics
↓
Not synchronization
```

---

# 23. Completion Checklist

- [x] `Thread.State` definition
- [x] Six official states
- [x] `RUNNABLE` deep dive
- [x] `RUNNABLE` vs OS-level READY/RUNNING
- [x] `start()` behavior
- [x] Scheduling nondeterminism
- [x] `RUNNABLE` vs `BLOCKED`
- [x] `RUNNABLE` vs `WAITING`
- [x] `RUNNABLE` vs `TIMED_WAITING`
- [x] `RUNNABLE` → `TERMINATED`
- [x] `getState()` snapshot behavior
- [x] Practice code
- [x] Practice exercises
- [x] Interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.4 — Thread Lifecycle](../04-Thread-Lifecycle/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.6 — Thread Naming & Basic Thread APIs**