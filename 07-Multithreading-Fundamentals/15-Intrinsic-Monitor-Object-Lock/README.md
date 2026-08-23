# 7.15 — Intrinsic Monitor / Object Lock

## 🎯 Objective

Understand Java's **intrinsic monitor**, how every object can be used as a monitor, how `synchronized` acquires/releases that monitor, and the relationship between object locks, `this`, class locks, and synchronized methods/blocks.

---

## 1. What is an Intrinsic Monitor? ⭐⭐⭐⭐⭐

Every Java object can participate in synchronization through an associated **intrinsic monitor**.

When a thread enters:

```java
synchronized (lock) {
    // critical section
}
```

it must acquire the monitor associated with `lock`.

Only one thread at a time can own a particular monitor for ordinary synchronized entry.

### Interview definition

> An intrinsic monitor is the built-in synchronization mechanism associated with an object. A `synchronized` method or block uses that monitor to provide mutual exclusion and the relevant memory-visibility guarantees.

---

# 2. Object as a Lock ⭐⭐⭐⭐⭐

A normal object can be used as a lock:

```java
Object lock = new Object();

synchronized (lock) {
    // protected code
}
```

The important thing is **object identity**, not the object's fields or value.

Two references referring to the same object use the same monitor.

```java
Object a = new Object();
Object b = a;
```

Here:

```java
synchronized (a) { }
synchronized (b) { }
```

use the same monitor because `a` and `b` refer to the same object.

---

# 3. Same Object = Same Monitor ⭐⭐⭐⭐⭐

```java
Object lock = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock) {
        // T1 owns lock's monitor
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock) {
        // T2 needs the same monitor
    }
});
```

If T1 owns the monitor, T2 cannot enter its synchronized block using that same monitor until T1 releases it.

### Golden rule

> **Same object identity → same intrinsic monitor.**

---

# 4. Different Objects = Different Monitors ⭐⭐⭐⭐⭐

```java
Object lock1 = new Object();
Object lock2 = new Object();
```

Then:

```java
synchronized (lock1) { }
synchronized (lock2) { }
```

use different monitors.

Synchronization on `lock1` does not automatically block synchronization on `lock2`.

### Interview trap

> `synchronized` is not a global lock. The monitor being acquired determines who competes with whom.

---

# 5. Practice Code — Same vs Different Lock

```java
public class MonitorIdentityPractice {

    private final Object lock = new Object();

    public void first() {
        synchronized (lock) {
            System.out.println("First entered");
            sleep();
            System.out.println("First leaving");
        }
    }

    public void second() {
        synchronized (lock) {
            System.out.println("Second entered");
        }
    }

    private void sleep() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        MonitorIdentityPractice object =
                new MonitorIdentityPractice();

        Thread t1 = new Thread(object::first);
        Thread t2 = new Thread(object::second);

        t1.start();
        Thread.sleep(100);
        t2.start();

        t1.join();
        t2.join();
    }
}
```

T2 has to wait because both methods synchronize on the same monitor.

---

# 6. `this` Is an Object Monitor ⭐⭐⭐⭐⭐

For an instance object:

```java
synchronized (this) {
    // protected code
}
```

locks the monitor associated with the current object.

Therefore:

```java
public synchronized void increment() {
    counter++;
}
```

uses the same object monitor as:

```java
public void increment() {
    synchronized (this) {
        counter++;
    }
}
```

for that instance method.

---

# 7. Practice Code — `this` Monitor

```java
public class ThisMonitorPractice {

    private int counter;

    public void increment() {
        synchronized (this) {
            counter++;
        }
    }

    public void printCounter() {
        synchronized (this) {
            System.out.println("Counter = " + counter);
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        ThisMonitorPractice object =
                new ThisMonitorPractice();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                object.increment();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                object.increment();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        object.printCounter();
    }
}
```

---

# 8. Synchronized Instance Method and Object Monitor ⭐⭐⭐⭐⭐

An instance synchronized method:

```java
public synchronized void update() {
    // ...
}
```

uses the monitor of the current object (`this`).

Conceptually:

```java
public void update() {
    synchronized (this) {
        // ...
    }
}
```

This means two threads calling synchronized instance methods on the **same object** compete for that object's monitor.

---

# 9. Different Instances Have Different Object Locks ⭐⭐⭐⭐⭐

Consider:

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

An instance synchronized method on `c1` locks `c1`'s monitor.

An instance synchronized method on `c2` locks `c2`'s monitor.

Therefore, calls on `c1` and `c2` can proceed concurrently unless another shared lock/state introduces coordination.

### Important

```text
c1 → monitor M1
c2 → monitor M2
```

They are different objects, so they have different intrinsic monitors.

---

# 10. Practice Code — Two Object Monitors

```java
public class TwoObjectLocksPractice {

    public synchronized void work(String name) {
        System.out.println(name + " entered");

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println(name + " leaving");
    }

    public static void main(String[] args)
            throws InterruptedException {

        TwoObjectLocksPractice first =
                new TwoObjectLocksPractice();
        TwoObjectLocksPractice second =
                new TwoObjectLocksPractice();

        Thread t1 = new Thread(() -> first.work("T1"));
        Thread t2 = new Thread(() -> second.work("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

The two calls use different object monitors, so they do not block each other merely because the method is synchronized.

---

# 11. Class Object as a Monitor ⭐⭐⭐⭐⭐

Every class has a corresponding `Class` object.

For example:

```java
MyService.class
```

can be used as a monitor:

```java
synchronized (MyService.class) {
    // static shared state
}
```

This is different from an instance monitor.

```text
instance monitor → object instance
class monitor    → Class object
```

---

# 12. Static Synchronized Method Uses Class Monitor ⭐⭐⭐⭐⭐

A static synchronized method:

```java
public static synchronized void update() {
    // ...
}
```

uses the monitor associated with the class object.

Conceptually:

```java
public static void update() {
    synchronized (MyService.class) {
        // ...
    }
}
```

### Memory trick

```text
synchronized instance method → this
synchronized static method   → Class object
```

---

# 13. Practice Code — Instance Lock vs Class Lock

```java
public class InstanceAndClassMonitorPractice {

    private int instanceCounter;
    private static int staticCounter;

    public synchronized void incrementInstance() {
        instanceCounter++;
    }

    public static synchronized void incrementStatic() {
        staticCounter++;
    }

    public static void main(String[] args)
            throws InterruptedException {

        InstanceAndClassMonitorPractice object =
                new InstanceAndClassMonitorPractice();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                object.incrementInstance();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                incrementStatic();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println("Instance = " + object.instanceCounter);
        System.out.println("Static   = " + staticCounter);
    }
}
```

The instance method and static method use different monitors.

---

# 14. Monitor Ownership ⭐⭐⭐⭐⭐

A thread that successfully enters a synchronized block becomes the owner of that monitor until it exits the synchronized region.

```java
synchronized (LOCK) {
    // current thread owns LOCK's monitor here
}
```

Another thread attempting to acquire the same monitor must wait until ownership is released.

### Important

The monitor is associated with the **lock object**, not with the thread itself.

---

# 15. Reentrant Nature of Intrinsic Monitors ⭐⭐⭐⭐⭐

Intrinsic monitors are **reentrant**.

If a thread already owns a monitor, it can enter another synchronized region protected by the same monitor without deadlocking itself.

Example:

```java
public synchronized void methodA() {
    methodB();
}

public synchronized void methodB() {
    System.out.println("Inside B");
}
```

The same thread can acquire the same object's monitor again.

The JVM keeps track of the monitor ownership/reentrance so the thread can eventually release it correctly.

A deeper treatment of reentrancy is covered in the later dedicated topic.

---

# 16. Practice Code — Reentrant Monitor

```java
public class ReentrantMonitorPractice {

    public synchronized void methodA() {
        System.out.println("Inside A");
        methodB();
    }

    public synchronized void methodB() {
        System.out.println("Inside B");
    }

    public static void main(String[] args) {
        ReentrantMonitorPractice object =
                new ReentrantMonitorPractice();

        object.methodA();
    }
}
```

Output:

```text
Inside A
Inside B
```

The thread does not deadlock when it re-enters the same monitor.

---

# 17. `synchronized` Does Not Lock the Variable ⭐⭐⭐⭐⭐

This is an important conceptual distinction.

```java
Object lock = new Object();
```

The monitor is associated with the **object**, not with the reference variable itself.

If:

```java
Object a = lock;
Object b = lock;
```

then `a` and `b` identify the same object and therefore the same monitor.

If:

```java
Object a = new Object();
Object b = new Object();
```

then they identify different objects and therefore different monitors.

---

# 18. Practice Code — Reference Identity

```java
public class ReferenceIdentityPractice {

    public static void main(String[] args) {

        Object first = new Object();
        Object second = first;
        Object third = new Object();

        System.out.println(first == second); // true
        System.out.println(first == third);  // false

        synchronized (first) {
            System.out.println("First monitor acquired");
        }

        synchronized (second) {
            System.out.println("Same monitor acquired again");
        }

        synchronized (third) {
            System.out.println("Different monitor acquired");
        }
    }
}
```

---

# 19. `wait()` and the Intrinsic Monitor ⭐⭐⭐⭐⭐

`wait()`, `notify()`, and `notifyAll()` are tied to intrinsic monitors.

A thread must own the object's monitor before calling these methods on that object:

```java
synchronized (LOCK) {
    LOCK.wait();
}
```

Calling `wait()` causes the thread to release that object's monitor while waiting.

This differs from:

```java
Thread.sleep(...);
```

which does not release a monitor the thread currently owns.

The complete `wait/notify` mechanism is covered later in Chapter 7.

---

# 20. Monitor vs `Lock` Interface ⭐⭐⭐⭐

Intrinsic monitor:

```java
synchronized (lock) {
    // ...
}
```

Explicit lock:

```java
Lock lock = new ReentrantLock();

lock.lock();
try {
    // ...
} finally {
    lock.unlock();
}
```

### Intrinsic monitor strengths

- Built into Java language
- Automatic release when leaving synchronized region
- Simple syntax
- Reentrant
- Strong memory semantics

### Explicit `Lock` strengths

- `tryLock()`
- interruptible acquisition
- optional fairness support in suitable implementations
- more advanced locking control

---

# 21. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Every object has one global JVM lock

❌ No. Each object has its own intrinsic monitor.

### Mistake 2 — Two references always mean two locks

❌ No. If both references point to the same object, they use the same monitor.

### Mistake 3 — Two objects of the same class share an instance monitor

❌ No. Each object has its own instance monitor.

### Mistake 4 — Static synchronized method locks `this`

❌ No. It uses the class object's monitor.

### Mistake 5 — `sleep()` releases the monitor

❌ No.

### Mistake 6 — `wait()` and `sleep()` are equivalent

❌ No. `wait()` releases the relevant monitor while waiting; `sleep()` does not.

### Mistake 7 — Intrinsic monitors are not reentrant

❌ False. They are reentrant.

### Mistake 8 — Locking on `new Object()` every time protects shared state

❌ Usually false because each expression creates a different object/monitor.

---

# 22. Interview Questions

### Q1. What is an intrinsic monitor?

The built-in synchronization mechanism associated with an object and used by `synchronized` blocks/methods.

### Q2. What object does an instance synchronized method lock?

The current object (`this`).

### Q3. What does a static synchronized method lock?

The monitor associated with the class object.

### Q4. Do two references to the same object use the same monitor?

Yes.

### Q5. Do two objects of the same class share the same instance monitor?

No. Each object has its own monitor.

### Q6. Are intrinsic monitors reentrant?

Yes.

### Q7. Does `sleep()` release an intrinsic monitor?

No.

### Q8. Does `wait()` release the object's monitor?

Yes, when invoked correctly while owning that monitor; the thread releases that monitor while waiting.

### Q9. Why does `synchronized (new Object())` fail as a shared lock pattern?

Because each evaluation creates a different object and therefore a different monitor.

### Q10. Is a monitor associated with a reference variable?

No. It is associated with the object identity to which the reference points.

### Q11. What is the difference between an instance lock and a class lock?

An instance lock is the monitor of a particular object; a class lock is the monitor of the corresponding `Class` object.

### Q12. Can two synchronized methods execute concurrently?

It depends on the monitors they use. If they use the same monitor, no; if they use different monitors, they may.

---

# 23. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"In Java, an intrinsic monitor is the built-in synchronization mechanism associated with an object. When a thread enters a synchronized block, it acquires the monitor of the object used in the synchronized expression. The key point is object identity: two references to the same object use the same monitor, while different objects have different monitors. An instance synchronized method uses the current object's monitor, `this`, whereas a static synchronized method uses the monitor of the corresponding Class object. Intrinsic monitors are reentrant, so a thread that already owns a monitor can acquire it again. Synchronization provides mutual exclusion and the relevant memory-visibility guarantees through monitor release and acquisition. `sleep()` does not release a monitor, while `wait()` releases the associated monitor while waiting. This monitor model is the foundation behind synchronized methods, synchronized blocks, and the classic wait/notify mechanism."**

---

# 24. Quick Revision

```text
Java Object
    ↓
Intrinsic Monitor
    ↓
Thread acquires monitor
    ↓
Synchronized code executes
    ↓
Thread releases monitor
```

### Lock Mapping ⭐⭐⭐⭐⭐

```text
synchronized instance method
        ↓
      this

synchronized(this)
        ↓
      this

synchronized static method
        ↓
   Class object

synchronized(MyClass.class)
        ↓
   Class object
```

### Golden Rules

```text
Same object      → same monitor
Different object → different monitor
Instance method  → this monitor
Static method    → Class monitor
Monitor          → reentrant
sleep()          → keeps monitor
wait()           → releases monitor while waiting
```

### Memory Trick

> **Monitor belongs to the object, not the variable.**

---

# 25. Completion Checklist

- [x] Intrinsic monitor definition
- [x] Object as a lock
- [x] Same object vs different object
- [x] Object identity
- [x] `this` monitor
- [x] Instance synchronized method
- [x] Different instance monitors
- [x] Class object monitor
- [x] Static synchronized method
- [x] Monitor ownership
- [x] Reentrancy basics
- [x] Reference vs object identity
- [x] `wait()` relationship
- [x] `sleep()` relationship
- [x] Intrinsic monitor vs `Lock`
- [x] Practice code
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.14 — `synchronized` Block](../14-Synchronized-Block/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.16 — Class Lock vs Object Lock**