# Chapter 7 — Multithreading Fundamentals

> **Status:** 🚧 In Progress  
> **Level:** 5+ Years Java Developer / Interview Preparation  
> **Approach:** Concept → Internal Working → Code → Edge Cases → Interview Questions → Revision

## 🎯 Chapter Goal

By the end of this chapter, you should be able to explain and implement Java threads confidently, understand synchronization and thread communication, and reason about race conditions, visibility, atomicity, deadlock, starvation and livelock.

---

# 1. Thread & Process Fundamentals ⭐⭐⭐⭐⭐

### 1.1 Process
- What is a process?
- Process memory isolation
- Process resources
- Process vs application

### 1.2 Thread
- What is a thread?
- Why threads are needed
- Thread shares process resources
- Thread stack vs process memory
- User thread vs daemon thread

### 1.3 Process vs Thread

| Process | Thread |
|---|---|
| Independent execution unit | Lightweight execution unit inside a process |
| Separate process address space | Shares process heap/resources |
| More expensive to create/switch | Generally cheaper to create/switch |
| IPC needed for communication | Shared memory enables communication |

### Interview Focus
- Why is a thread called lightweight?
- What memory is shared between threads?
- What memory is private to each thread?

---

# 2. Creating Threads ⭐⭐⭐⭐⭐

### 2.1 Extending `Thread`

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

public class ThreadCreationDemo {
    public static void main(String[] args) {
        Thread thread = new MyThread();
        thread.start();
    }
}
```

### 2.2 Implementing `Runnable`

```java
class Task implements Runnable {
    @Override
    public void run() {
        System.out.println("Task executed by " + Thread.currentThread().getName());
    }
}

public class RunnableDemo {
    public static void main(String[] args) {
        Thread thread = new Thread(new Task());
        thread.start();
    }
}
```

### 2.3 Why Prefer `Runnable`?

- Java supports single class inheritance.
- Extending `Thread` consumes the inheritance slot.
- Separates **task** from **thread**.
- Makes the task reusable with executors later.

### Critical Interview Point

```java
thread.start(); // creates/schedules execution of a new thread
thread.run();   // ordinary method call; does NOT create a new thread
```

---

# 3. Thread Lifecycle ⭐⭐⭐⭐⭐

Java exposes these `Thread.State` values:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

### State Flow

```text
NEW
 ↓ start()
RUNNABLE
 ├── BLOCKED
 ├── WAITING
 ├── TIMED_WAITING
 └── TERMINATED
```

### Important Nuance

Java's `Thread.State` does **not** define separate `RUNNING` and `READY` states. Both are represented by `RUNNABLE`; the exact scheduler/OS behavior is implementation-dependent.

### Important Methods

- `start()`
- `run()`
- `getState()`
- `isAlive()`
- `interrupt()`

---

# 4. Thread Naming & Basic APIs

### Important APIs

```java
Thread.currentThread()
thread.getName()
thread.setName("worker-1")
thread.getId()
thread.getPriority()
thread.setPriority(...)
thread.isAlive()
thread.isDaemon()
thread.setDaemon(true)
```

### Priority

Thread priority is only a scheduling hint; application correctness must never depend on it.

### Daemon Thread

A daemon thread does not keep the JVM alive once all non-daemon threads have terminated.

---

# 5. `sleep()` ⭐⭐⭐⭐⭐

```java
Thread.sleep(1000);
```

### Understand

- Puts current thread into `TIMED_WAITING`.
- Does not release an intrinsic monitor lock already held by the thread.
- Throws `InterruptedException`.
- Sleep duration is not an exact execution guarantee.

### Interview Comparison

| `sleep()` | `wait()` |
|---|---|
| `Thread` method | `Object` method |
| Timed pause | Coordination/waiting mechanism |
| Does not release monitor | Releases monitor when waiting |
| Can be used without owning a monitor | Must call while owning that object's monitor |

---

# 6. `join()` ⭐⭐⭐⭐⭐

```java
Thread worker = new Thread(task);
worker.start();
worker.join();
```

The calling thread waits for the target thread to terminate.

### Variants

```java
join()
join(long millis)
join(long millis, int nanos)
```

### Interview Question

**Who waits when `main` calls `worker.join()`?**

`main` waits for `worker`; the worker does not wait for `main` because of that call.

---

# 7. `yield()` ⭐⭐⭐⭐

```java
Thread.yield();
```

### Key Facts

- Scheduler hint only.
- Does not guarantee another thread runs.
- Does not release a held monitor lock.
- Leaves the Java thread in `RUNNABLE` state.
- Portable application logic should not depend on it.

---

# 8. Synchronization Fundamentals ⭐⭐⭐⭐⭐

### 8.1 Race Condition

A race condition occurs when multiple threads access shared mutable state and the final result depends on timing/interleaving.

Example:

```java
counter++;
```

Conceptually:

```text
read counter
add 1
write counter
```

Two threads can interleave these operations and lose an update.

### 8.2 Critical Section

A critical section is code that accesses shared state and must satisfy the application's synchronization requirements.

### 8.3 Thread Safety

A component is thread-safe when its behavior remains correct under the allowed concurrent access patterns.

---

# 9. `synchronized` ⭐⭐⭐⭐⭐

### 9.1 Synchronized Instance Method

```java
public synchronized void increment() {
    counter++;
}
```

Locks the monitor associated with the current object.

### 9.2 Synchronized Static Method

```java
public static synchronized void increment() {
    counter++;
}
```

Locks the `Class` object's monitor for that class.

### 9.3 Synchronized Block

```java
public void increment() {
    synchronized (this) {
        counter++;
    }
}
```

Prefer a narrowly scoped lock when appropriate.

### 9.4 Lock Identity

```text
synchronized instance method → this object's monitor
synchronized static method   → Class object's monitor
synchronized(lockObject)     → lockObject's monitor
```

---

# 10. Intrinsic Monitor / Lock ⭐⭐⭐⭐⭐

Every Java object can be associated with an intrinsic monitor.

A thread entering a `synchronized` region must acquire the relevant monitor.

### Important Properties

- Mutual exclusion for synchronized regions using the same monitor.
- Reentrant: the same thread can acquire the same monitor again.
- Monitor ownership matters for `wait()`, `notify()` and `notifyAll()`.

### Reentrancy Example

```java
public synchronized void outer() {
    inner();
}

public synchronized void inner() {
    System.out.println("Reentrant acquisition");
}
```

The same thread can enter `inner()` while already holding the same object's monitor.

---

# 11. Visibility, Atomicity & Ordering ⭐⭐⭐⭐⭐

### Visibility

One thread must be able to observe another thread's relevant state updates according to the Java Memory Model.

### Atomicity

An operation is atomic when it appears indivisible with respect to the relevant concurrent observers.

Example:

```java
counter++;
```

is not generally an atomic read-modify-write operation.

### Ordering

The Java Memory Model defines which executions and reorderings are legal and how synchronization establishes ordering relationships.

### Three Core Questions

```text
Can another thread SEE the update?     → Visibility
Can the operation be split/interleaved? → Atomicity
What execution order is guaranteed?     → Ordering
```

---

# 12. Happens-Before ⭐⭐⭐⭐⭐

Understand the Java Memory Model through happens-before relationships.

Important examples include:

- Unlock of a monitor happens-before a subsequent lock of the same monitor.
- A write to a `volatile` variable happens-before a subsequent read of that same variable.
- A call to `Thread.start()` happens-before actions in the started thread.
- All actions in a thread happen-before another thread successfully returns from `join()` on that thread.

This topic connects synchronization, visibility and ordering.

---

# 13. `wait()` / `notify()` / `notifyAll()` ⭐⭐⭐⭐⭐

### `wait()`

```java
synchronized (lock) {
    lock.wait();
}
```

The calling thread waits and releases that object's monitor while waiting.

### `notify()`

```java
synchronized (lock) {
    lock.notify();
}
```

Requests that one waiting thread be awakened.

### `notifyAll()`

```java
synchronized (lock) {
    lock.notifyAll();
}
```

Requests that all waiting threads be awakened.

### Critical Rule

`wait()`, `notify()` and `notifyAll()` must be invoked while the current thread owns the relevant object's monitor; otherwise `IllegalMonitorStateException` occurs.

### Correct Waiting Pattern

Use a condition loop:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // condition is satisfied
}
```

Do not rely on a single `if` check because wakeups can occur without the condition being true.

---

# 14. `wait()` vs `sleep()` vs `join()` ⭐⭐⭐⭐⭐

| Feature | `wait()` | `sleep()` | `join()` |
|---|---|---|---|
| Declared by | `Object` | `Thread` | `Thread` |
| Primary purpose | Thread coordination | Timed pause | Wait for another thread |
| Releases monitor? | Yes, the waited object's monitor | No | No, unless separate application synchronization causes otherwise |
| Needs monitor ownership? | Yes | No | No |
| InterruptedException | Yes | Yes | Yes |

---

# 15. Interrupts ⭐⭐⭐⭐⭐

```java
thread.interrupt();
```

An interrupt is a cooperative cancellation/notification mechanism, not a forced thread kill.

### Important APIs

```java
isInterrupted()
Thread.interrupted()
interrupt()
```

### Difference

- `isInterrupted()` checks the current interrupt status without clearing it.
- `Thread.interrupted()` checks the current thread's status and clears it.

### Best Practice

If a method catches `InterruptedException` but cannot rethrow it, it should normally restore the interrupt status:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

# 16. Thread Communication Pattern ⭐⭐⭐⭐⭐

Classic producer-consumer coordination can be expressed using a shared condition and monitor:

```text
Producer
   ↓
shared state
   ↓
notify / notifyAll
   ↓
Consumer
   ↓
wait while condition is false
```

For production systems, higher-level concurrency utilities are generally preferable and are covered in Chapter 8.

---

# 17. Deadlock ⭐⭐⭐⭐⭐

Deadlock occurs when threads are permanently blocked waiting for locks/resources held by one another.

### Four Coffman Conditions

1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

### Example Pattern

```text
Thread-1 holds Lock-A → waits for Lock-B
Thread-2 holds Lock-B → waits for Lock-A
```

### Prevention

- Consistent lock ordering
- Reduce lock scope
- Avoid nested locks where possible
- Use timed lock acquisition with explicit lock APIs when appropriate
- Avoid calling unknown/external code while holding locks

---

# 18. Starvation ⭐⭐⭐⭐

A thread experiences starvation when it repeatedly fails to obtain the CPU or required resources/locks and therefore makes insufficient progress.

### Possible Causes

- Unfair resource allocation
- Long-running lock holders
- Excessive priority differences
- Poor synchronization design

---

# 19. Livelock ⭐⭐⭐⭐

Threads remain active and keep responding to one another, but no useful progress occurs.

```text
Thread A changes state
        ↕
Thread B changes state
        ↕
Both continue reacting
        ↓
No progress
```

### Deadlock vs Livelock

```text
Deadlock → blocked, no progress
Livelock  → active, but no useful progress
```

---

# 20. Common Thread-Safety Strategies ⭐⭐⭐⭐⭐

### Strategy 1 — Immutability

Prefer immutable shared objects where possible.

### Strategy 2 — Thread Confinement

Keep mutable state confined to one thread.

### Strategy 3 — Synchronization

Protect shared mutable state with a correct synchronization policy.

### Strategy 4 — Atomic Variables

Covered more deeply in Chapter 8.

### Strategy 5 — Concurrent Collections

Covered in Chapter 8.

### Strategy 6 — Message Passing

Reduce direct shared mutable state and communicate through safe queues/abstractions.

---

# 21. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1
Calling `run()` expecting a new thread.

### Mistake 2
Assuming `sleep()` releases a monitor.

### Mistake 3
Calling `wait()` without owning the object's monitor.

### Mistake 4
Using `if` instead of a condition loop around `wait()`.

### Mistake 5
Assuming `volatile` makes `counter++` atomic.

### Mistake 6
Using `Thread.stop()` as normal cancellation logic.

### Mistake 7
Assuming `yield()` guarantees another thread will run.

### Mistake 8
Assuming thread priority provides correctness guarantees.

### Mistake 9
Holding locks while calling slow or unknown external code.

### Mistake 10
Ignoring interruption and swallowing `InterruptedException`.

---

# 22. Interview Comparison Matrix ⭐⭐⭐⭐⭐

| Topic | Key Difference |
|---|---|
| Process vs Thread | Process is independent; thread executes within a process |
| `start()` vs `run()` | `start()` enables new-thread execution; `run()` is a normal call |
| `sleep()` vs `wait()` | Pause vs coordination; `wait()` releases waited monitor |
| `wait()` vs `notify()` | Waits for condition vs signals waiting threads |
| `notify()` vs `notifyAll()` | Wake one waiter vs request wakeup of all waiters |
| `synchronized` vs `volatile` | Mutual exclusion/visibility vs visibility/order semantics; volatile does not provide compound-operation atomicity |
| Deadlock vs starvation | Circular permanent blocking vs insufficient progress due to resource access |
| Deadlock vs livelock | Blocked with no progress vs active but no useful progress |

---

# 23. Practical Coding Problems ⭐⭐⭐⭐⭐

Implement and explain:

1. Create two threads using `Thread`.
2. Create a thread using `Runnable`.
3. Print odd/even numbers using two threads.
4. Implement a thread-safe counter.
5. Implement producer-consumer using `wait()`/`notifyAll()`.
6. Demonstrate a race condition.
7. Fix the race condition using `synchronized`.
8. Demonstrate deadlock.
9. Fix deadlock using lock ordering.
10. Coordinate worker completion using `join()`.
11. Demonstrate interruption correctly.
12. Demonstrate the difference between `run()` and `start()`.

---

# 24. Interview Scenarios ⭐⭐⭐⭐⭐

### Scenario 1
**Two threads increment a shared counter 1,000 times. Why can the result be less than 2,000?**

Because `counter++` is a compound read-modify-write operation and concurrent updates can be lost.

### Scenario 2
**Why doesn't `volatile int counter` solve `counter++`?**

`volatile` provides visibility and ordering guarantees for the variable, but does not make the compound increment operation atomic.

### Scenario 3
**Why must `wait()` be called inside synchronized code?**

Because the thread must own the relevant object's monitor when invoking `wait()`; the wait operation is tied to that monitor's condition queue.

### Scenario 4
**Does `yield()` release a lock?**

No. It is a scheduler hint and does not release a held monitor.

### Scenario 5
**What happens if two threads synchronize on different objects?**

They do not mutually exclude one another merely because both blocks use `synchronized`; the same monitor object must be used for mutual exclusion over the same shared resource.

---

# 25. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **“A Java thread is an execution path inside a process. I can create one by extending Thread or, preferably, by implementing Runnable because it separates the task from the execution mechanism. Java exposes NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING and TERMINATED states; RUNNABLE covers both ready-to-run and running behavior at the Java API level. When multiple threads share mutable state, I need to reason about visibility, atomicity and ordering. `synchronized` provides monitor-based mutual exclusion and establishes visibility guarantees. `wait()` is used for monitor-based coordination and releases the waited object's monitor, while `sleep()` only pauses the current thread and does not release its monitor. `join()` waits for another thread to finish, and `yield()` is only a scheduling hint. I also need to handle interruption correctly and understand deadlock, starvation and livelock. For higher-level thread management, executors and concurrency utilities are covered separately.”**

---

# 26. Quick Revision Sheet

```text
PROCESS
  ↓
THREAD
  ↓
start() → RUNNABLE
  ↓
Synchronization
  ├── synchronized
  ├── monitor
  └── thread safety
  ↓
Communication
  ├── wait()
  ├── notify()
  └── notifyAll()
  ↓
Thread Control
  ├── sleep()
  ├── join()
  ├── interrupt()
  └── yield()
  ↓
JMM Concepts
  ├── visibility
  ├── atomicity
  ├── ordering
  └── happens-before
  ↓
Failure Modes
  ├── deadlock
  ├── starvation
  └── livelock
```

---

# 27. Completion Checklist

- [ ] Process vs Thread
- [ ] Thread creation
- [ ] `Thread` class
- [ ] `Runnable`
- [ ] `start()` vs `run()`
- [ ] Thread lifecycle
- [ ] `Thread.State`
- [ ] Thread naming / daemon / priority
- [ ] `sleep()`
- [ ] `join()`
- [ ] `yield()`
- [ ] Race condition
- [ ] Critical section
- [ ] Thread safety
- [ ] `synchronized` method
- [ ] `synchronized` block
- [ ] Intrinsic monitor
- [ ] Reentrancy
- [ ] Visibility
- [ ] Atomicity
- [ ] Ordering
- [ ] Happens-before
- [ ] `wait()`
- [ ] `notify()`
- [ ] `notifyAll()`
- [ ] Interrupts
- [ ] Deadlock
- [ ] Starvation
- [ ] Livelock
- [ ] Practical coding problems
- [ ] Interview scenarios
- [ ] 2-minute answer
- [ ] Quick revision

---

## Navigation

[🏠 Core Java Master README](../README.md)

**Next → Chapter 8 — Multithreading Enhancements / Concurrency Utilities**

**Chapter 7 subtopics will be completed one-by-one from this roadmap.**