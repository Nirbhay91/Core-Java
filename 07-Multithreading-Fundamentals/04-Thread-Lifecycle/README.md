# 7.4 — Thread Lifecycle

## 🎯 Objective

Understand the complete lifecycle of a Java `Thread`, the states defined by `Thread.State`, how a thread moves between states, and the interview traps around `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, and `TERMINATED`.

---

## 1. What is Thread Lifecycle?

A Java thread does not remain in one state throughout its lifetime. Its state changes based on operations such as `start()`, lock acquisition, `wait()`, `sleep()`, `join()`, interruption, and completion of `run()`.

High-level flow:

```text
NEW
 ↓ start()
RUNNABLE
 ↓
Execution / scheduling
 ↓
 ┌───────────────┐
 │               │
 │ BLOCKED       │ ← waiting for monitor lock
 │ WAITING       │ ← waiting indefinitely
 │ TIMED_WAITING │ ← waiting for a bounded time
 │               │
 └───────┬───────┘
         ↓
      RUNNABLE
         ↓
    run() completes
         ↓
    TERMINATED
```

---

## 2. Java's Official Thread States ⭐⭐⭐⭐⭐

Java defines six states in `Thread.State`:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

These are available through:

```java
Thread.State
```

Example:

```java
Thread t = new Thread(() -> {
    System.out.println("Running");
});

System.out.println(t.getState());
```

Before `start()` the expected state is:

```text
NEW
```

---

## 3. NEW State

A thread is in `NEW` after its `Thread` object has been created but before `start()` has been invoked.

```java
Thread t = new Thread(() -> {
    System.out.println("Hello");
});

System.out.println(t.getState());
```

Output:

```text
NEW
```

Important:

```text
new Thread(...)
       ↓
     NEW
```

The thread has not started execution yet.

---

## 4. `start()` Transition

When `start()` is called:

```java
t.start();
```

the thread becomes eligible for execution and its state becomes `RUNNABLE` according to the Java thread-state model.

Conceptually:

```text
NEW
 ↓ start()
RUNNABLE
```

The exact moment at which the thread gets CPU time is not controlled by the Java application.

---

## 5. RUNNABLE State ⭐⭐⭐⭐⭐

`RUNNABLE` is the Java state representing a thread that is eligible to run and may be actually executing or waiting for processor availability.

This is an important interview point:

> Java does not expose separate `RUNNING` and `READY` states in `Thread.State`.

So avoid saying:

```text
RUNNING → READY
```

as official Java thread states.

Instead:

```text
RUNNABLE
```

covers the Java-level state.

Example:

```java
Thread t = new Thread(() -> {
    for (int i = 0; i < 100000; i++) {
        Math.sqrt(i);
    }
});

t.start();
```

During its execution, its Java-level state can be observed as `RUNNABLE`.

---

## 6. BLOCKED State ⭐⭐⭐⭐⭐

A thread enters `BLOCKED` when it is waiting to acquire an intrinsic monitor lock in order to enter a `synchronized` block or method.

Example:

```java
class LockDemo {

    synchronized void work() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

If one thread already owns the monitor and another thread tries to enter the synchronized method:

```text
Thread A
   ↓
owns monitor
   ↓
inside synchronized method

Thread B
   ↓
tries to acquire same monitor
   ↓
BLOCKED
```

### Key point

`BLOCKED` specifically relates to waiting for a monitor lock.

---

## 7. WAITING State ⭐⭐⭐⭐⭐

A thread enters `WAITING` when it waits indefinitely for another thread to perform an action.

Common operations that can cause `WAITING`:

```java
Object.wait()
Thread.join()
LockSupport.park()
```

Examples:

```java
thread.join();
```

or:

```java
synchronized (lock) {
    lock.wait();
}
```

Conceptually:

```text
RUNNABLE
   ↓
wait() / join() / park()
   ↓
WAITING
   ↓
notification / joined thread completion / unpark
   ↓
RUNNABLE
```

---

## 8. TIMED_WAITING State ⭐⭐⭐⭐⭐

A thread enters `TIMED_WAITING` when it waits for a specified maximum amount of time.

Common operations:

```java
Thread.sleep(...)
Object.wait(timeout)
Thread.join(timeout)
LockSupport.parkNanos(...)
LockSupport.parkUntil(...)
```

Example:

```java
Thread.sleep(1000);
```

Conceptually:

```text
RUNNABLE
   ↓
sleep(1000)
   ↓
TIMED_WAITING
   ↓
time expires
   ↓
RUNNABLE
```

### Important

`sleep()` does **not** release an intrinsic monitor lock held by the sleeping thread.

This is a frequent interview question.

---

## 9. TERMINATED State

A thread enters `TERMINATED` after its execution completes.

Example:

```java
Thread t = new Thread(() -> {
    System.out.println("Task completed");
});

t.start();
```

After `run()` finishes:

```text
RUNNABLE
   ↓
run() completes
   ↓
TERMINATED
```

A terminated thread cannot be restarted.

---

## 10. Complete Lifecycle Example

```java
public class ThreadLifecycleDemo {

    public static void main(String[] args)
            throws InterruptedException {

        Thread t = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        System.out.println("1. " + t.getState());

        t.start();
        System.out.println("2. " + t.getState());

        Thread.sleep(100);
        System.out.println("3. " + t.getState());

        t.join();
        System.out.println("4. " + t.getState());
    }
}
```

Typical conceptual progression:

```text
NEW
 ↓
RUNNABLE
 ↓
TIMED_WAITING
 ↓
RUNNABLE
 ↓
TERMINATED
```

The exact observation timing is not deterministic, so a real program should not assume a particular state at an arbitrary instant.

---

## 11. State Transition Map ⭐⭐⭐⭐⭐

```text
                    start()
             ┌──────────────────┐
             │                  ↓
          NEW ──────────────> RUNNABLE
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ↓                ↓                ↓
             BLOCKED          WAITING      TIMED_WAITING
                │                │                │
                └────────────────┼────────────────┘
                                 ↓
                              RUNNABLE
                                 │
                          run() completes
                                 ↓
                            TERMINATED
```

This is a conceptual model. The actual transition depends on the operation and synchronization context.

---

## 12. BLOCKED vs WAITING ⭐⭐⭐⭐⭐

This distinction is extremely important.

### BLOCKED

The thread wants to acquire an intrinsic monitor lock.

```text
Thread A owns lock
       ↓
Thread B wants same lock
       ↓
BLOCKED
```

### WAITING

The thread is waiting indefinitely for another thread/action.

```text
wait()
join()
park()
       ↓
WAITING
```

### Interview answer

> `BLOCKED` means the thread is waiting to acquire a monitor lock. `WAITING` means the thread is waiting indefinitely for another thread or synchronization action.

---

## 13. WAITING vs TIMED_WAITING

### WAITING

No timeout.

```java
lock.wait();
thread.join();
LockSupport.park();
```

### TIMED_WAITING

Has a timeout.

```java
Thread.sleep(1000);
lock.wait(1000);
thread.join(1000);
```

### Remember

```text
WAITING        → indefinite
TIMED_WAITING  → bounded wait
```

---

## 14. `sleep()` and Thread State

When a thread executes:

```java
Thread.sleep(2000);
```

it enters `TIMED_WAITING` for the requested duration, subject to scheduling and interruption.

Important:

```text
sleep()
  ↓
TIMED_WAITING
```

And:

```text
sleep() does NOT release monitor locks
```

---

## 15. `wait()` and Thread State

When a thread calls:

```java
lock.wait();
```

while owning the object's monitor, it releases that monitor and enters `WAITING` until notified/interrupted or otherwise transitioned according to the waiting mechanism.

Example:

```java
synchronized (lock) {
    lock.wait();
}
```

Conceptually:

```text
Own monitor
   ↓
wait()
   ↓
Release monitor
   ↓
WAITING
```

When awakened, the thread must reacquire the monitor before it can continue past `wait()`.

---

## 16. `join()` and Thread State

If thread A executes:

```java
threadB.join();
```

thread A waits for thread B to terminate.

A can enter `WAITING` for an untimed `join()` or `TIMED_WAITING` for a timed `join(timeout)`.

```text
A
 ↓
join(B)
 ↓
WAITING
 ↓
B terminates
 ↓
A becomes RUNNABLE
```

---

## 17. Interrupt and Lifecycle

Calling:

```java
thread.interrupt();
```

does not forcibly kill the thread.

If a thread is blocked in interruptible methods such as `sleep()`, `wait()`, or `join()`, an `InterruptedException` may be thrown and the thread can then decide how to respond.

For a normal running thread, interruption generally sets its interrupt status; the thread's code must cooperate by checking or responding to it.

Example:

```java
try {
    Thread.sleep(5000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

## 18. Can We Force a Thread into a State?

No general-purpose API exists to safely force a thread into arbitrary lifecycle states.

For example, you cannot reliably say:

```text
make thread BLOCKED now
```

The state results from actual synchronization and lifecycle operations.

Use:

```java
thread.getState();
```

to observe the current Java-level state.

---

## 19. `getState()` Caveat ⭐⭐⭐⭐⭐

```java
Thread.State state = thread.getState();
```

The returned state is a snapshot.

Another thread may change state immediately after the call.

Therefore, this is useful for:

- Debugging
- Monitoring
- Diagnostics

but should not normally be used as a synchronization mechanism.

---

## 20. `NEW` vs `TERMINATED`

Both are states where the thread is not executing.

But they mean very different things:

```text
NEW
 ↓
Not started yet

TERMINATED
 ↓
Execution already completed
```

A `NEW` thread can be started once.

A `TERMINATED` thread cannot be restarted.

---

## 21. Important Interview Trap: RUNNING State

Question:

> What are the seven states of a Java thread?

Correct answer:

There are **six** states in Java's `Thread.State` enum:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

There is no separate Java API state called `RUNNING`.

Operating systems may distinguish ready/running internally, but Java exposes them through `RUNNABLE`.

---

## 22. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Saying Java has RUNNING state

❌ Incorrect.

Java's official state is `RUNNABLE`.

### Mistake 2 — Saying `sleep()` causes `WAITING`

❌ Incorrect.

`sleep()` causes `TIMED_WAITING`.

### Mistake 3 — Saying `wait()` keeps the monitor lock

❌ Incorrect.

`wait()` releases the object's monitor while waiting.

### Mistake 4 — Saying `sleep()` releases the monitor

❌ Incorrect.

`sleep()` does not release intrinsic monitor locks.

### Mistake 5 — Treating `getState()` as synchronization

❌ Incorrect.

The state is only a snapshot.

### Mistake 6 — Assuming exact state timing

❌ Incorrect.

Thread scheduling is nondeterministic, so observations can vary.

---

## 23. Interview Questions

### Q1. What are the six Java thread states?

`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, and `TERMINATED`.

### Q2. What state is a thread in after construction but before `start()`?

`NEW`.

### Q3. What happens after `start()`?

The thread becomes eligible for execution and enters the Java-level `RUNNABLE` state.

### Q4. What is BLOCKED?

Waiting to acquire an intrinsic monitor lock.

### Q5. What is WAITING?

Waiting indefinitely for another thread/action, commonly through `wait()`, untimed `join()`, or `park()`.

### Q6. What is TIMED_WAITING?

Waiting for a specified maximum duration, commonly through `sleep()`, timed `wait()`, or timed `join()`.

### Q7. Does Java have a RUNNING state?

No. Java exposes `RUNNABLE`; actual CPU scheduling distinctions are not represented as separate `Thread.State` values.

### Q8. Difference between BLOCKED and WAITING?

`BLOCKED` is waiting for a monitor lock; `WAITING` is waiting indefinitely for a synchronization/event action.

### Q9. Difference between `sleep()` and `wait()` regarding locks?

`sleep()` does not release an intrinsic monitor. `wait()` releases the object's monitor while waiting.

### Q10. Can a terminated thread be restarted?

No. A new `Thread` instance is required for another execution.

---

## 24. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"A Java thread has six official states: NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, and TERMINATED. A newly created thread is NEW. Calling `start()` makes it eligible for execution and it enters RUNNABLE. BLOCKED means it is waiting for an intrinsic monitor lock. WAITING means it is waiting indefinitely, for example through `wait()` or an untimed `join()`. TIMED_WAITING means it is waiting for a bounded duration, such as during `sleep()` or a timed `join()`. When `run()` completes, the thread becomes TERMINATED. An important interview point is that Java does not expose a separate RUNNING state; both ready and actually executing situations are represented by RUNNABLE at the Java API level." 

---

## 25. Quick Revision

```text
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
   run() completes
        ↓
   TERMINATED
```

### Six States

```text
1. NEW
2. RUNNABLE
3. BLOCKED
4. WAITING
5. TIMED_WAITING
6. TERMINATED
```

### Must Remember

```text
BLOCKED       → waiting for monitor lock
WAITING       → indefinite wait
TIMED_WAITING → bounded wait
sleep()       → TIMED_WAITING
wait()        → WAITING + releases monitor
join()        → WAITING / TIMED_WAITING
```

```text
No Java RUNNING state.
RUNNABLE is the Java-level state for both ready/executing situations.
```

---

## 26. Completion Checklist

- [x] Thread lifecycle overview
- [x] Six official Java states
- [x] NEW
- [x] RUNNABLE
- [x] BLOCKED
- [x] WAITING
- [x] TIMED_WAITING
- [x] TERMINATED
- [x] State transition model
- [x] `start()` transition
- [x] `sleep()` state
- [x] `wait()` state and monitor release
- [x] `join()` state
- [x] Interrupt connection
- [x] `getState()` caveat
- [x] RUNNABLE vs OS scheduling terminology
- [x] Common interview traps
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.3 — Thread Creation using `Runnable`](../03-Thread-Creation-Runnable/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.5 — `Thread.State` and `RUNNABLE`**