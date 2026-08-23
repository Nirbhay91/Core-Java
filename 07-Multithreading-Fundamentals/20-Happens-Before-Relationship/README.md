# 7.20 — Happens-Before Relationship

## 🎯 Objective

Understand the **Java Memory Model (JMM)** happens-before relationship and use it to reason about **visibility, ordering and safe communication between threads**.

> **Interview rule:** Happens-before is a formal ordering guarantee. It is not simply "Thread A physically executes before Thread B".

---

# 1. What is Happens-Before? ⭐⭐⭐⭐⭐

If action **A happens-before B**, then the Java Memory Model guarantees that A is ordered before B and that the effects required by the memory model are visible to B.

```text
Action A
   |
   | happens-before
   v
Action B
```

### Easy definition

> **Happens-before is a JMM relationship that establishes ordering and visibility guarantees between actions.**

It does **not** mean that A must finish in real time before B starts in every physical execution scenario; it defines what observations are guaranteed by the memory model.

---

# 2. Why Do We Need It? ⭐⭐⭐⭐⭐

Without a formal memory model, multithreaded reasoning would be unreliable because:

- Compilers can optimize code.
- CPUs can execute operations out of order internally.
- Memory systems can delay visibility of writes.
- Threads can observe shared state differently without synchronization.

The JMM defines synchronization and ordering rules that make concurrent programs reason-able.

---

# 3. The Most Important Happens-Before Rules ⭐⭐⭐⭐⭐

Remember these rules for interviews:

```text
1. Program Order Rule
2. Monitor Lock Rule
3. Volatile Variable Rule
4. Thread Start Rule
5. Thread Termination / Join Rule
6. Thread Interruption Rule
7. Transitivity Rule
```

---

# 4. Program Order Rule ⭐⭐⭐⭐⭐

Within a single thread, each action happens-before every subsequent action in that thread according to program order.

Example:

```java
int a = 10;
int b = 20;
int c = a + b;
```

The earlier actions are ordered before later actions in that thread.

```text
a = 10
  ↓
b = 20
  ↓
c = a + b
```

### Important

This does not mean every low-level machine instruction is executed exactly in source-code order. The JMM defines the ordering guarantees that matter for observable behavior.

---

# 5. Practice Code — Program Order

```java
public class ProgramOrderDemo {

    public static void main(String[] args) {
        int value = 10;

        value = value + 5;
        System.out.println(value);
    }
}
```

For a single thread, actions are governed by program order.

---

# 6. Monitor Lock Rule ⭐⭐⭐⭐⭐

For the **same monitor**:

```text
unlock
  ↓ happens-before
subsequent lock
```

Example:

```java
synchronized (lock) {
    value = 42;
}
```

Another thread later enters:

```java
synchronized (lock) {
    System.out.println(value);
}
```

The unlock of `lock` happens-before the subsequent lock of the same monitor.

This is one reason `synchronized` provides both **mutual exclusion** and **memory visibility/ordering guarantees**.

---

# 7. Practice Code — Monitor Happens-Before ⭐⭐⭐⭐⭐

```java
public class MonitorHappensBeforeDemo {

    private final Object lock = new Object();
    private int value;

    public void writer() {
        synchronized (lock) {
            value = 42;
        }
    }

    public void reader() {
        synchronized (lock) {
            System.out.println("Value = " + value);
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        MonitorHappensBeforeDemo demo =
                new MonitorHappensBeforeDemo();

        Thread writer = new Thread(demo::writer);
        Thread reader = new Thread(demo::reader);

        writer.start();
        writer.join();

        reader.start();
        reader.join();
    }
}
```

The synchronization on the same monitor provides the relevant happens-before relationship.

---

# 8. Volatile Variable Rule ⭐⭐⭐⭐⭐

A write to a `volatile` variable happens-before every subsequent read of that same variable.

```java
volatile boolean ready;
```

Writer:

```java
ready = true;
```

Reader:

```java
if (ready) {
    // observe state published before the volatile write
}
```

### Memory Trick

```text
volatile write
      ↓
   happens-before
      ↓
volatile read
```

---

# 9. Practice Code — Volatile Happens-Before ⭐⭐⭐⭐⭐

```java
public class VolatileHappensBeforeDemo {

    private int data;
    private volatile boolean ready;

    public void writer() {
        data = 42;
        ready = true;
    }

    public void reader() {
        if (ready) {
            System.out.println("Data = " + data);
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        VolatileHappensBeforeDemo demo =
                new VolatileHappensBeforeDemo();

        Thread writer = new Thread(demo::writer);
        Thread reader = new Thread(demo::reader);

        writer.start();
        writer.join();

        reader.start();
        reader.join();
    }
}
```

The important relationship is:

```text
data = 42
    ↓
ready = true  (volatile write)
    ↓
happens-before
volatile read of ready
    ↓
reader can rely on preceding publication under the JMM
```

---

# 10. Thread Start Rule ⭐⭐⭐⭐⭐

A call to:

```java
thread.start();
```

happens-before any actions in the started thread.

Therefore, actions performed by the parent thread before `start()` are ordered before the started thread's actions.

---

# 11. Practice Code — `start()` Happens-Before ⭐⭐⭐⭐⭐

```java
public class StartHappensBeforeDemo {

    private int value;

    public static void main(String[] args)
            throws InterruptedException {

        StartHappensBeforeDemo demo =
                new StartHappensBeforeDemo();

        demo.value = 100;

        Thread worker = new Thread(() -> {
            System.out.println("Worker sees = " + demo.value);
        });

        worker.start();
        worker.join();
    }
}
```

Conceptually:

```text
value = 100
     ↓
start()
     ↓ happens-before
worker actions
```

---

# 12. Thread Termination / `join()` Rule ⭐⭐⭐⭐⭐

Actions in a thread happen-before another thread successfully returns from:

```java
thread.join();
```

This makes `join()` a useful synchronization point when waiting for a worker's result.

---

# 13. Practice Code — `join()` Happens-Before ⭐⭐⭐⭐⭐

```java
public class JoinHappensBeforeDemo {

    public static void main(String[] args)
            throws InterruptedException {

        final int[] result = new int[1];

        Thread worker = new Thread(() -> {
            result[0] = 500;
        });

        worker.start();
        worker.join();

        System.out.println("Result = " + result[0]);
    }
}
```

Conceptually:

```text
worker writes result
       ↓
worker terminates
       ↓
join() returns
       ↓
main reads result
```

The successful return from `join()` establishes the required happens-before relationship.

---

# 14. Thread Interruption Rule ⭐⭐⭐⭐

An invocation of:

```java
thread.interrupt();
```

happens-before the interrupted thread detects the interrupt through the relevant interruption mechanism, such as `InterruptedException` or interrupt-status checks as specified by the API/JMM.

### Important

Interrupt is a **cooperative signal**, not a forced thread termination mechanism.

---

# 15. Practice Code — Interrupt Coordination

```java
public class InterruptHappensBeforeDemo {

    public static void main(String[] args)
            throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(10_000);
            } catch (InterruptedException e) {
                System.out.println("Worker received interrupt");
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        Thread.sleep(100);
        worker.interrupt();

        worker.join();
    }
}
```

The example demonstrates cooperative interruption and `join()`-based completion coordination.

---

# 16. Transitivity Rule ⭐⭐⭐⭐⭐

If:

```text
A happens-before B
B happens-before C
```

then:

```text
A happens-before C
```

This is **transitivity**.

---

# 17. Practice Code — Transitivity

A useful conceptual chain is:

```text
Thread A
   |
   | writes data
   v
volatile flag write
   |
   | happens-before
   v
Thread B volatile flag read
   |
   | program order
   v
Thread B reads data
```

The happens-before relationship can be reasoned about through the chain of ordering relationships.

---

# 18. Happens-Before vs Execution Order ⭐⭐⭐⭐⭐

This is a common interview trap.

### Wrong

> "Happens-before means A physically executes completely before B starts."

### Correct

> "Happens-before is a Java Memory Model ordering relationship that provides guarantees about the ordering and visibility of actions."

It is a formal concurrency rule, not a simple timestamp.

---

# 19. Happens-Before vs Synchronization ⭐⭐⭐⭐⭐

Synchronization mechanisms can establish happens-before relationships.

Examples:

```text
synchronized
volatile
Thread.start()
Thread.join()
interrupt-related rules
```

But not every arbitrary interaction between threads creates a happens-before relationship.

---

# 20. Why `sleep()` Is NOT a General Visibility Mechanism ⭐⭐⭐⭐⭐

This is another interview trap.

```java
Thread.sleep(1000);
```

does **not** by itself create a general happens-before relationship between arbitrary writes in one thread and reads in another.

Therefore, this is not a correct synchronization strategy:

```java
while (!ready) {
    Thread.sleep(100);
}
```

if `ready` is not otherwise safely published.

Use an appropriate synchronization mechanism such as:

```java
volatile
synchronized
Lock
CountDownLatch
Future
```

as appropriate to the design.

---

# 21. Practice Code — Correct Flag Publication

```java
public class SafeFlagDemo {

    private volatile boolean ready;

    public void writer() {
        ready = true;
    }

    public void reader() {
        while (!ready) {
            // Wait for publication of the flag.
        }

        System.out.println("Ready observed");
    }
}
```

The `volatile` variable provides the required visibility/order semantics for this simple flag.

---

# 22. Happens-Before and `synchronized` ⭐⭐⭐⭐⭐

Consider:

```java
synchronized (lock) {
    data = 42;
}
```

followed by:

```java
synchronized (lock) {
    System.out.println(data);
}
```

The relevant relationship is:

```text
Thread A
  data = 42
       ↓
  unlock(lock)
       ↓ happens-before
  lock(lock)
       ↓
Thread B
  read data
```

The **same monitor** is important.

---

# 23. Happens-Before and `volatile` ⭐⭐⭐⭐⭐

```java
private int data;
private volatile boolean ready;
```

Writer:

```java
data = 42;
ready = true;
```

Reader:

```java
if (ready) {
    System.out.println(data);
}
```

Reasoning:

```text
data write
   ↓ program order
volatile ready write
   ↓ happens-before
volatile ready read
   ↓ program order
reader's data read
```

This is a classic publication pattern.

---

# 24. Happens-Before and `start()` + `join()` ⭐⭐⭐⭐⭐

A useful interview chain:

```text
Main writes configuration
        ↓
Thread.start()
        ↓
Worker executes
        ↓
Worker writes result
        ↓
Worker terminates
        ↓
Main returns from join()
        ↓
Main reads result
```

The start and join relationships allow the threads to safely communicate in this pattern.

---

# 25. Happens-Before Does NOT Mean Atomicity ⭐⭐⭐⭐⭐

A happens-before relationship can provide visibility and ordering without making a compound operation atomic.

For example, `volatile` can establish a happens-before relationship:

```java
volatile int count;
```

but:

```java
count++;
```

is still a compound read-modify-write operation and is not made atomic merely by `volatile`.

---

# 26. Happens-Before Does NOT Mean Lock-Free ⭐⭐⭐⭐

A program can use synchronization and therefore establish happens-before relationships while still using locks.

Happens-before describes memory/order guarantees; it does not classify an implementation as lock-free, blocking or non-blocking.

---

# 27. Happens-Before Quick Reference ⭐⭐⭐⭐⭐

| Rule | Happens-Before Relationship |
|---|---|
| Program Order | Earlier action in a thread → later action in same thread |
| Monitor | Unlock of monitor → subsequent lock of same monitor |
| Volatile | Write to volatile → subsequent read of same volatile |
| Start | Actions before `start()` → actions in started thread |
| Join | Actions in thread → successful return from `join()` |
| Interrupt | `interrupt()` invocation → corresponding detection/handling as specified |
| Transitivity | A → B and B → C implies A → C |

---

# 28. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**Does `sleep()` establish visibility?**

No. Sleep is not a general synchronization mechanism.

### Trap 2
**Does `volatile` make `count++` atomic?**

No.

### Trap 3
**Does happens-before mean wall-clock order?**

No. It is a JMM relationship.

### Trap 4
**Does `join()` wait for the current thread?**

The calling thread waits for the target thread on which `join()` was invoked.

### Trap 5
**Does every shared-memory access create happens-before?**

No. The program needs a rule that establishes the relationship.

### Trap 6
**Does synchronized only prevent two threads from entering together?**

No. It also provides memory visibility and ordering guarantees through monitor synchronization.

### Trap 7
**Is volatile a replacement for all synchronization?**

No. It is suitable for specific visibility/order requirements but does not provide general mutual exclusion or compound-operation atomicity.

---

# 29. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is happens-before?

A formal JMM relationship that establishes ordering and visibility guarantees between actions.

### Q2. Name important happens-before rules.

Program order, monitor unlock/lock, volatile write/read, `start()`, `join()`, interrupt-related ordering, and transitivity.

### Q3. What happens-before relationship does `start()` provide?

Actions before `start()` happen-before actions in the started thread.

### Q4. What happens-before relationship does `join()` provide?

Actions in the target thread happen-before a successful return from `join()` in the joining thread.

### Q5. Does `sleep()` create happens-before?

No, not as a general synchronization mechanism.

### Q6. Does volatile provide happens-before?

Yes. A write to a volatile variable happens-before a subsequent read of that same variable.

### Q7. Does synchronized provide happens-before?

Yes. An unlock of a monitor happens-before a subsequent lock of the same monitor.

### Q8. What is transitivity?

If A happens-before B and B happens-before C, then A happens-before C.

### Q9. Does happens-before guarantee physical execution order?

No. It defines memory-model ordering/visibility guarantees.

### Q10. Is happens-before the same as atomicity?

No. Happens-before primarily gives ordering and visibility guarantees; atomicity concerns indivisible operations.

---

# 30. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Happens-before is a formal relationship defined by the Java Memory Model. If action A happens-before action B, the JMM provides the required ordering and visibility guarantees between those actions. Important happens-before rules include program order within a thread, monitor unlock followed by a subsequent lock on the same monitor, a volatile write followed by a subsequent read of the same volatile variable, actions before `Thread.start()` happening-before actions in the started thread, and actions in a thread happening-before another thread successfully returns from `join()`. The relationship is transitive, so if A happens-before B and B happens-before C, then A happens-before C. A key interview point is that happens-before does not simply mean physical wall-clock execution order, and it does not by itself mean that a compound operation is atomic. It is one of the core tools for reasoning about visibility and ordering in concurrent Java programs."**

---

# 31. Quick Revision ⭐⭐⭐⭐⭐

```text
Happens-Before
      ↓
JMM ordering + visibility guarantee

Program Order
A → B in same thread

Monitor
unlock → subsequent lock on same monitor

Volatile
volatile write → subsequent read of same variable

start()
before start → started thread actions

join()
worker actions → successful join return

Transitivity
A → B → C  =>  A → C

sleep()
❌ not a general synchronization mechanism

volatile
❌ does not make count++ atomic

Happens-before
❌ not simply wall-clock execution order
```

### Golden Rule

> **When you see a multithreading visibility question, ask: "What happens-before relationship makes this observation guaranteed?"**

---

# 32. Practice Checklist

- [x] Definition of happens-before
- [x] Why JMM needs happens-before
- [x] Program order
- [x] Monitor unlock/lock
- [x] Volatile write/read
- [x] `Thread.start()`
- [x] `Thread.join()`
- [x] Interrupt ordering
- [x] Transitivity
- [x] `sleep()` misconception
- [x] Volatile publication pattern
- [x] Synchronization publication pattern
- [x] Start + join communication pattern
- [x] Atomicity vs happens-before
- [x] Common interview traps
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.19 — Atomicity vs Visibility vs Ordering](../19-Atomicity-Visibility-Ordering/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.21 — `volatile` Fundamentals**