# 7.13 — `synchronized` Method

## 🎯 Objective

Understand how Java's `synchronized` method works, which monitor it locks, instance vs static synchronized methods, reentrancy, visibility, common mistakes, and practical interview scenarios.

---

## 1. What is a `synchronized` Method? ⭐⭐⭐⭐⭐

A method declared with the `synchronized` keyword allows only one thread at a time to execute that method for the **same monitor**.

```java
public synchronized void increment() {
    counter++;
}
```

For an instance method, the monitor is the current object (`this`).

For a static synchronized method, the monitor is the `Class` object.

### Interview definition

> A synchronized method uses an intrinsic monitor to provide mutual exclusion for the method's execution and establishes the relevant Java Memory Model happens-before relationship when the same monitor is used for synchronization.

---

# 2. Instance `synchronized` Method ⭐⭐⭐⭐⭐

```java
class Counter {

    private int count;

    public synchronized void increment() {
        count++;
    }
}
```

Conceptually equivalent to:

```java
class Counter {

    private int count;

    public void increment() {
        synchronized (this) {
            count++;
        }
    }
}
```

For an instance synchronized method:

```text
synchronized method
       ↓
lock this object
       ↓
execute method
       ↓
release monitor
```

---

# 3. Same Object vs Different Objects ⭐⭐⭐⭐⭐

This is one of the most important interview points.

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

If two threads call:

```java
c1.increment();
c1.increment();
```

they contend for the **same** `c1` monitor.

But:

```java
c1.increment();
c2.increment();
```

use different object monitors and therefore do not block each other merely because the method is synchronized.

### Memory trick

```text
Same object    → same instance monitor
Different object → different instance monitors
```

---

# 4. Practice Code — Same Object ⭐⭐⭐⭐⭐

```java
public class SameObjectPractice {

    static class Counter {
        private int count;

        public synchronized void increment() {
            for (int i = 0; i < 3; i++) {
                System.out.println(
                        Thread.currentThread().getName()
                                + " -> " + ++count);
            }
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        Counter counter = new Counter();

        Thread first = new Thread(counter::increment, "Thread-A");
        Thread second = new Thread(counter::increment, "Thread-B");

        first.start();
        second.start();

        first.join();
        second.join();
    }
}
```

Both threads use the same object, so only one can hold that object's monitor at a time.

---

# 5. Practice Code — Different Objects ⭐⭐⭐⭐⭐

```java
public class DifferentObjectsPractice {

    static class Counter {

        public synchronized void work() {
            System.out.println(
                    Thread.currentThread().getName()
                            + " entered");

            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println(
                    Thread.currentThread().getName()
                            + " leaving");
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        Counter c1 = new Counter();
        Counter c2 = new Counter();

        Thread first = new Thread(c1::work, "Thread-A");
        Thread second = new Thread(c2::work, "Thread-B");

        first.start();
        second.start();

        first.join();
        second.join();
    }
}
```

The methods are synchronized, but they lock different objects.

---

# 6. Static `synchronized` Method ⭐⭐⭐⭐⭐

```java
class Counter {

    private static int count;

    public static synchronized void increment() {
        count++;
    }
}
```

Conceptually equivalent to:

```java
class Counter {

    private static int count;

    public static void increment() {
        synchronized (Counter.class) {
            count++;
        }
    }
}
```

The monitor is the `Class` object:

```text
Counter.class
```

---

# 7. Instance vs Static `synchronized` ⭐⭐⭐⭐⭐

| Method | Monitor |
|---|---|
| `public synchronized void m()` | current object (`this`) |
| `public static synchronized void m()` | class object (`MyClass.class`) |

### Important

An instance synchronized method and a static synchronized method do **not** use the same monitor merely because they belong to the same class.

Conceptually:

```text
instance method → this
static method   → MyClass.class
```

---

# 8. Practice Code — Instance vs Static Lock ⭐⭐⭐⭐⭐

```java
public class InstanceVsStaticPractice {

    public synchronized void instanceWork() {
        System.out.println("Instance method started");

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("Instance method finished");
    }

    public static synchronized void staticWork() {
        System.out.println("Static method started");

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("Static method finished");
    }

    public static void main(String[] args)
            throws InterruptedException {

        InstanceVsStaticPractice object =
                new InstanceVsStaticPractice();

        Thread first = new Thread(
                object::instanceWork,
                "Instance-Thread");

        Thread second = new Thread(
                InstanceVsStaticPractice::staticWork,
                "Static-Thread");

        first.start();
        second.start();

        first.join();
        second.join();
    }
}
```

They use different monitors and therefore do not automatically block one another.

---

# 9. Synchronized Method and Mutual Exclusion ⭐⭐⭐⭐⭐

For the same object:

```java
public synchronized void method() {
    // critical section
}
```

only one thread at a time can execute code protected by that object's monitor.

But this does **not** mean:

> "Only one thread can execute any method of this object."

For example:

```java
public synchronized void update() { }

public void read() { }
```

A thread can execute `read()` without acquiring the object's monitor unless `read()` itself performs synchronization or otherwise follows a synchronization protocol.

---

# 10. Two Synchronized Instance Methods ⭐⭐⭐⭐⭐

```java
class Account {

    public synchronized void deposit() {
        // ...
    }

    public synchronized void withdraw() {
        // ...
    }
}
```

For the **same object**, both methods use the same `this` monitor.

Therefore, two threads cannot simultaneously execute these synchronized methods on that same object.

```text
Thread A → deposit() → locks this
Thread B → withdraw() → waits for this
```

---

# 11. Practice Code — Two Synchronized Methods

```java
public class TwoMethodsPractice {

    static class Account {

        private int balance = 1000;

        public synchronized void deposit(int amount) {
            balance += amount;
            System.out.println("Deposit: " + balance);
        }

        public synchronized void withdraw(int amount) {
            if (balance >= amount) {
                balance -= amount;
                System.out.println("Withdraw: " + balance);
            }
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        Account account = new Account();

        Thread depositThread = new Thread(
                () -> account.deposit(500));

        Thread withdrawThread = new Thread(
                () -> account.withdraw(300));

        depositThread.start();
        withdrawThread.start();

        depositThread.join();
        withdrawThread.join();
    }
}
```

Both operations use the same `this` monitor.

---

# 12. Reentrancy ⭐⭐⭐⭐⭐

Java's intrinsic monitors are **reentrant**.

If a thread already owns an object's monitor, it can enter another synchronized method/block protected by that same monitor without deadlocking itself.

Example:

```java
class Example {

    public synchronized void outer() {
        inner();
    }

    public synchronized void inner() {
        System.out.println("Inside inner");
    }
}
```

A thread holding `this` while executing `outer()` can enter `inner()` because it already owns the same monitor.

The detailed topic of reentrancy will be covered separately in **7.17 — Reentrancy of `synchronized`**.

---

# 13. Practice Code — Reentrant Synchronized Methods

```java
public class ReentrantPractice {

    public synchronized void outer() {
        System.out.println("outer() entered");
        inner();
        System.out.println("outer() leaving");
    }

    public synchronized void inner() {
        System.out.println("inner() entered");
    }

    public static void main(String[] args) {
        new ReentrantPractice().outer();
    }
}
```

This does not deadlock because the same thread already owns the monitor.

---

# 14. `synchronized` Does Two Important Things ⭐⭐⭐⭐⭐

It is useful to think about two major properties:

### 1. Mutual exclusion

For the same monitor, only one thread can execute the protected synchronized region at a time.

### 2. Memory visibility / happens-before

An unlock on a monitor happens-before a subsequent lock on that same monitor.

This helps establish visibility of changes made while holding the monitor to a thread that later acquires the same monitor.

### Interview line

> `synchronized` provides both mutual exclusion and important memory-visibility guarantees through the Java Memory Model.

---

# 15. `synchronized` Is Not Just About Performance

Do not describe synchronization only as:

> "It prevents two threads from entering at the same time."

That is incomplete.

It also establishes memory-ordering/visibility guarantees through monitor acquire/release semantics.

---

# 16. Synchronized Method vs Synchronized Block ⭐⭐⭐⭐⭐

### Method

```java
public synchronized void update() {
    counter++;
}
```

### Block

```java
public void update() {
    synchronized (this) {
        counter++;
    }
}
```

A synchronized block gives more control over:

- synchronization scope
- lock object
- amount of code protected

A synchronized method is concise and useful when the whole method needs the same monitor.

---

# 17. Practice Code — Same Behavior

```java
class Counter {

    private int count;

    public synchronized void incrementUsingMethod() {
        count++;
    }

    public void incrementUsingBlock() {
        synchronized (this) {
            count++;
        }
    }
}
```

For the instance monitor, both use `this`.

They are not identical in every possible refactoring because a block can use a different lock object and can protect only a subset of the method.

---

# 18. What Does a Synchronized Method Lock? ⭐⭐⭐⭐⭐

### Instance

```java
public synchronized void method() {}
```

locks:

```java
this
```

### Static

```java
public static synchronized void method() {}
```

locks:

```java
MyClass.class
```

### Memory trick

```text
No static → this
static    → Class.class
```

---

# 19. Does `synchronized` Lock the Method? ⭐⭐⭐⭐⭐

❌ No.

It does not lock the method itself.

It acquires a monitor associated with an object or class object.

Correct mental model:

```text
synchronized instance method
        ↓
acquire this monitor
        ↓
execute method
        ↓
release monitor
```

---

# 20. What Happens When a Thread Calls a Synchronized Method?

Conceptually:

```text
Thread calls method
        ↓
Attempts monitor acquisition
        ↓
Monitor available?
   ↙             ↘
 yes              no
  ↓                ↓
enter          wait to acquire
  ↓
execute
  ↓
exit
  ↓
release monitor
```

The exact JVM scheduling/implementation details are more complex, but this is the correct interview-level model.

---

# 21. Practice Code — Contention ⭐⭐⭐⭐⭐

```java
public class ContentionPractice {

    public synchronized void work() {
        System.out.println(
                Thread.currentThread().getName()
                        + " acquired monitor");

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println(
                Thread.currentThread().getName()
                        + " releasing monitor");
    }

    public static void main(String[] args)
            throws InterruptedException {

        ContentionPractice object =
                new ContentionPractice();

        Thread first = new Thread(object::work, "T1");
        Thread second = new Thread(object::work, "T2");

        first.start();
        second.start();

        first.join();
        second.join();
    }
}
```

The second thread cannot execute `work()` concurrently on the same object while the first thread owns its monitor.

---

# 22. `sleep()` Inside a Synchronized Method ⭐⭐⭐⭐⭐

Consider:

```java
public synchronized void work() {
    Thread.sleep(1000);
}
```

While sleeping, the thread **does not release the object's monitor merely because it is sleeping**.

Therefore another thread trying to enter another synchronized method on the same object can remain blocked until the monitor is released.

### Important distinction

```text
sleep() → does not release monitor
wait()  → releases monitor when waiting on that monitor
```

---

# 23. Practice Code — `sleep()` Holds the Monitor

```java
public class SleepWithMonitorPractice {

    public synchronized void first() {
        System.out.println("first() entered");

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("first() leaving");
    }

    public synchronized void second() {
        System.out.println("second() entered");
    }

    public static void main(String[] args)
            throws InterruptedException {

        SleepWithMonitorPractice object =
                new SleepWithMonitorPractice();

        Thread first = new Thread(object::first, "T1");
        Thread second = new Thread(object::second, "T2");

        first.start();
        Thread.sleep(100);
        second.start();

        first.join();
        second.join();
    }
}
```

`T2` must wait for `T1` to release the monitor.

---

# 24. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — synchronized method locks the method

❌ It locks the relevant monitor.

### Mistake 2 — all objects of the class share one instance lock

❌ Each object has its own instance monitor.

### Mistake 3 — static and instance synchronized methods use the same lock

❌ Instance uses `this`; static uses the class object.

### Mistake 4 — `sleep()` releases the monitor

❌ It does not.

### Mistake 5 — synchronized means no other method can execute

❌ Unsynchronized methods can still execute unless they independently coordinate access.

### Mistake 6 — `volatile` is equivalent to synchronized

❌ It is not.

### Mistake 7 — synchronization only provides mutual exclusion

❌ It also provides important memory-visibility/happens-before guarantees.

### Mistake 8 — synchronization on different objects coordinates threads

❌ Different monitors do not provide mutual exclusion with each other.

---

# 25. Interview Questions

### Q1. What is a synchronized method?

A method whose execution is protected by an intrinsic monitor associated with the relevant object or class object.

### Q2. What lock does an instance synchronized method use?

The current object (`this`).

### Q3. What lock does a static synchronized method use?

The corresponding `Class` object.

### Q4. Can two threads execute the same synchronized instance method simultaneously?

They can if they invoke it on different objects; they cannot both hold the same object's monitor simultaneously.

### Q5. Can a synchronized and unsynchronized method execute simultaneously on the same object?

Yes, unless the unsynchronized method itself uses compatible synchronization. `synchronized` does not automatically block every method of the object.

### Q6. What is the difference between synchronized method and synchronized block?

A method protects the method using its intrinsic monitor; a block allows more precise scope and can choose the lock object.

### Q7. Is `synchronized` reentrant?

Yes. Java intrinsic monitors are reentrant.

### Q8. Does `sleep()` release a monitor?

No.

### Q9. Does synchronized provide visibility?

Yes. Monitor unlock/lock establishes the relevant happens-before relationship for the same monitor.

### Q10. Does `synchronized` guarantee fairness?

No. Java's intrinsic monitor does not provide a general fairness guarantee.

### Q11. Why might synchronization reduce performance?

Because lock contention can serialize execution and increase waiting/coordination overhead.

### Q12. Is synchronization always bad for performance?

No. Correctness comes first, and modern JVMs optimize many synchronization paths. Optimize based on measured contention/performance rather than avoiding synchronization blindly.

---

# 26. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"A synchronized method is a Java method protected by an intrinsic monitor. For an instance synchronized method, the monitor is the current object, `this`; for a static synchronized method, the monitor is the corresponding `Class` object. If two threads call synchronized instance methods on the same object, they cannot simultaneously own that object's monitor, so the protected execution is mutually exclusive. However, synchronized methods on different objects use different instance monitors and can execute concurrently. `synchronized` also provides important memory-visibility and happens-before guarantees; it is not only about mutual exclusion. A synchronized method can be replaced conceptually by a synchronized block on the same monitor, although blocks provide finer control over scope and lock choice. Java intrinsic monitors are reentrant, and `sleep()` does not release the monitor. In practice, we use synchronized methods when the whole method needs the same object's lock and use synchronized blocks when we need a narrower critical section or a specific lock object."**

---

# 27. Quick Revision

```text
synchronized instance method
          ↓
      lock this
          ↓
   one owner at a time
          ↓
 mutual exclusion
          +
 memory visibility
```

### Static

```text
static synchronized
        ↓
   lock Class object
```

### Golden Rules

```text
instance synchronized → this
static synchronized   → Class object
same object            → same instance monitor
different objects      → different monitors
sleep()                → does NOT release monitor
synchronized           → reentrant
synchronized           → mutual exclusion + visibility/happens-before
```

### Memory Trick

> **Instance → object lock. Static → class lock.**

---

# 28. Completion Checklist

- [x] Synchronized method definition
- [x] Instance synchronized method
- [x] Static synchronized method
- [x] `this` monitor
- [x] `Class` monitor
- [x] Same object vs different objects
- [x] Multiple synchronized methods
- [x] Reentrancy basics
- [x] Mutual exclusion
- [x] Visibility / happens-before
- [x] Method vs block
- [x] `sleep()` and monitor behavior
- [x] Contention practice
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.12 — Critical Section](../12-Critical-Section/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.14 — `synchronized` Block**