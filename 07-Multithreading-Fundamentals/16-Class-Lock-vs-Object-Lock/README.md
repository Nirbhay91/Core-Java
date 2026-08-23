# 7.16 — Class Lock vs Object Lock

## 🎯 Objective

Understand the difference between **object-level locking** and **class-level locking**, when each monitor is used, whether multiple instances can run concurrently, and how instance and static synchronization behave in real interview scenarios.

---

## 1. Core Idea ⭐⭐⭐⭐⭐

Java synchronization can use two fundamentally different monitors:

```text
Object Lock
    ↓
monitor of a particular object instance

Class Lock
    ↓
monitor of the Class object
```

### Object lock

```java
synchronized (this) {
    // object-level protection
}
```

or:

```java
public synchronized void method() {
    // object lock
}
```

### Class lock

```java
synchronized (MyClass.class) {
    // class-level protection
}
```

or:

```java
public static synchronized void method() {
    // class lock
}
```

---

# 2. Object Lock ⭐⭐⭐⭐⭐

An object lock is the intrinsic monitor associated with a specific object instance.

```java
MyService service = new MyService();
```

For an instance synchronized method:

```java
service.doWork();
```

the monitor of `service` is acquired.

If another thread tries to acquire the same object's monitor, it must wait until the first thread releases it.

### Memory trick

> **Instance method → instance object lock.**

---

# 3. Class Lock ⭐⭐⭐⭐⭐

A class lock refers to the monitor of the `Class` object.

```java
synchronized (MyService.class) {
    // class-level critical section
}
```

A static synchronized method also uses that class monitor:

```java
public static synchronized void update() {
    // class lock
}
```

### Memory trick

> **Static method → class object lock.**

---

# 4. Direct Comparison ⭐⭐⭐⭐⭐

| Feature | Object Lock | Class Lock |
|---|---|---|
| Monitor | Specific object | `Class` object |
| Typical method | Instance `synchronized` | Static `synchronized` |
| Syntax | `synchronized(this)` | `synchronized(MyClass.class)` |
| Scope | Per object instance | Per class |
| Different instances | Can use different locks | Same class monitor |
| Typical state | Instance state | Static/shared class state |

---

# 5. Multiple Instances Have Different Object Locks ⭐⭐⭐⭐⭐

Suppose:

```java
Service first = new Service();
Service second = new Service();
```

Then:

```text
first  → Object Monitor A
second → Object Monitor B
```

Therefore:

```java
first.work();
second.work();
```

can execute concurrently if `work()` is an instance synchronized method.

They are not competing for the same object monitor.

---

# 6. Practice Code — Different Object Locks ⭐⭐⭐⭐⭐

```java
public class ObjectLockPractice {

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

        ObjectLockPractice first = new ObjectLockPractice();
        ObjectLockPractice second = new ObjectLockPractice();

        Thread t1 = new Thread(() -> first.work("T1"));
        Thread t2 = new Thread(() -> second.work("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Observation

Both threads can enter around the same time because:

```text
first  → monitor 1
second → monitor 2
```

---

# 7. Same Object Means Same Object Lock ⭐⭐⭐⭐⭐

```java
Service service = new Service();
```

If two threads call:

```java
service.work();
```

and `work()` is an instance synchronized method, both compete for the same monitor.

```text
T1 ──→ service monitor ←── T2
```

Only one can own that monitor at a time.

---

# 8. Practice Code — Same Object Lock

```java
public class SameObjectLockPractice {

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

        SameObjectLockPractice service =
                new SameObjectLockPractice();

        Thread t1 = new Thread(() -> service.work("T1"));
        Thread t2 = new Thread(() -> service.work("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Observation

The calls are serialized because both use the same object's monitor.

---

# 9. Static Synchronized Method = Class Lock ⭐⭐⭐⭐⭐

Consider:

```java
public static synchronized void update() {
    // static shared state
}
```

This does **not** lock `this`.

It locks the monitor of the class object:

```java
MyClass.class
```

Conceptually:

```java
public static void update() {
    synchronized (MyClass.class) {
        // static shared state
    }
}
```

---

# 10. Practice Code — Class Lock

```java
public class ClassLockPractice {

    public static synchronized void work(String name) {
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

        Thread t1 = new Thread(() -> work("T1"));
        Thread t2 = new Thread(() -> work("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

Both threads use the same class monitor, so the static synchronized method is mutually exclusive across calls from different instances/threads.

---

# 11. Different Instances + Static Method ⭐⭐⭐⭐⭐

This is a common interview question.

```java
Service first = new Service();
Service second = new Service();
```

Even though `first` and `second` have different object monitors, a static synchronized method uses:

```java
Service.class
```

Therefore:

```text
first instance lock  ≠ Service.class lock
second instance lock ≠ Service.class lock
```

Both static calls still compete for the **same class lock**.

---

# 12. Practice Code — Two Objects, One Class Lock

```java
public class TwoObjectsOneClassLockPractice {

    public static synchronized void staticWork(String name) {
        System.out.println(name + " entered staticWork");

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println(name + " leaving staticWork");
    }

    public static void main(String[] args)
            throws InterruptedException {

        TwoObjectsOneClassLockPractice first =
                new TwoObjectsOneClassLockPractice();
        TwoObjectsOneClassLockPractice second =
                new TwoObjectsOneClassLockPractice();

        Thread t1 = new Thread(() ->
                TwoObjectsOneClassLockPractice.staticWork("T1"));

        Thread t2 = new Thread(() ->
                TwoObjectsOneClassLockPractice.staticWork("T2"));

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Key point

Creating multiple objects does **not** create multiple class locks.

There is one relevant `Class` object for the loaded class in a given class-loader context, and the static synchronized method uses that class monitor.

---

# 13. Instance Lock and Class Lock Are Independent ⭐⭐⭐⭐⭐

Consider:

```java
public synchronized void instanceWork() {
    // this monitor
}

public static synchronized void staticWork() {
    // MyClass.class monitor
}
```

These use different monitors.

Therefore, a thread holding the instance monitor does not automatically block a thread trying to acquire the class monitor.

```text
instanceWork()
     ↓
object monitor

staticWork()
     ↓
class monitor
```

---

# 14. Practice Code — Object Lock vs Class Lock Together ⭐⭐⭐⭐⭐

```java
public class IndependentLocksPractice {

    public synchronized void instanceWork() {
        System.out.println("Instance lock acquired");
        sleep();
        System.out.println("Instance work done");
    }

    public static synchronized void staticWork() {
        System.out.println("Class lock acquired");
        sleep();
        System.out.println("Static work done");
    }

    private static void sleep() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        IndependentLocksPractice object =
                new IndependentLocksPractice();

        Thread t1 = new Thread(object::instanceWork);
        Thread t2 = new Thread(
                IndependentLocksPractice::staticWork);

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```

### Interview answer

They can execute concurrently because they use different monitors.

---

# 15. Explicit Class Lock in a Block ⭐⭐⭐⭐⭐

A static critical section can also explicitly use the class object:

```java
public void updateStaticState() {
    synchronized (MyClass.class) {
        // protect static state
    }
}
```

This is useful when only a portion of a method needs the class-level lock.

---

# 16. Explicit Object Lock in a Block ⭐⭐⭐⭐⭐

```java
private final Object lock = new Object();

public void update() {
    synchronized (lock) {
        // protect instance state
    }
}
```

This is an object-level monitor that is separate from `MyClass.class`.

---

# 17. Practice Code — Both Explicit Locks

```java
public class ExplicitObjectAndClassLockPractice {

    private final Object objectLock = new Object();
    private static int staticCounter;
    private int instanceCounter;

    public void updateInstance() {
        synchronized (objectLock) {
            instanceCounter++;
        }
    }

    public void updateStatic() {
        synchronized (ExplicitObjectAndClassLockPractice.class) {
            staticCounter++;
        }
    }

    public static void main(String[] args) {
        ExplicitObjectAndClassLockPractice object =
                new ExplicitObjectAndClassLockPractice();

        object.updateInstance();
        object.updateStatic();

        System.out.println("Instance = " + object.instanceCounter);
        System.out.println("Static   = " + staticCounter);
    }
}
```

---

# 18. Important Interview Scenario ⭐⭐⭐⭐⭐

### Question

If one thread holds an object's lock and another thread calls a static synchronized method of the same class, will the second thread block?

### Answer

**Not because of those locks alone.**

The first thread holds:

```text
object monitor
```

The second thread wants:

```text
class monitor
```

These are different monitors.

So they do not automatically block each other.

---

# 19. Another Interview Scenario ⭐⭐⭐⭐⭐

### Question

Two different objects call a static synchronized method. Can they execute simultaneously?

### Answer

No, not with respect to that static synchronized method, because both calls use the same class monitor.

```text
Object A ─┐
          ├──→ MyClass.class monitor
Object B ─┘
```

---

# 20. Another Interview Scenario ⭐⭐⭐⭐⭐

### Question

Two different objects call an instance synchronized method. Can they execute simultaneously?

### Answer

Yes, because each instance has a different object monitor.

```text
Object A → Monitor A
Object B → Monitor B
```

Unless they share some other lock or shared synchronization mechanism.

---

# 21. Class Lock Does Not Mean All Instance Methods Are Locked ⭐⭐⭐⭐⭐

A class lock only coordinates threads that try to acquire that **same class monitor**.

For example:

```java
public static synchronized void staticWork() {
}
```

uses the class monitor.

But:

```java
public synchronized void instanceWork() {
}
```

uses an object monitor.

A thread inside `instanceWork()` does not automatically prevent another thread from entering `staticWork()`.

---

# 22. Practice Code — Static vs Instance Synchronization

```java
public class StaticVsInstancePractice {

    public synchronized void instanceWork() {
        System.out.println("Instance start");
        sleep();
        System.out.println("Instance end");
    }

    public static synchronized void staticWork() {
        System.out.println("Static start");
        sleep();
        System.out.println("Static end");
    }

    private static void sleep() {
        try {
            Thread.sleep(1500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args)
            throws InterruptedException {

        StaticVsInstancePractice object =
                new StaticVsInstancePractice();

        Thread instanceThread = new Thread(
                object::instanceWork);

        Thread staticThread = new Thread(
                StaticVsInstancePractice::staticWork);

        instanceThread.start();
        staticThread.start();

        instanceThread.join();
        staticThread.join();
    }
}
```

The two synchronized methods use independent monitors.

---

# 23. Lock Mapping Diagram ⭐⭐⭐⭐⭐

```text
                    Java Class
                        │
          ┌─────────────┴─────────────┐
          │                           │
     Class Object                 Instances
          │                           │
   MyClass.class               ┌──────┴──────┐
          │                    │             │
     Class Monitor           obj1          obj2
          │                    │             │
   static synchronized      Monitor 1     Monitor 2
```

### Therefore

```text
static synchronized → Class Monitor
instance synchronized → Object Monitor
```

---

# 24. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1

> Static synchronized method locks `this`.

❌ Wrong. Static methods have no `this`; the class object's monitor is used.

### Mistake 2

> Two instances share the same object lock.

❌ Wrong. Each instance has its own object monitor.

### Mistake 3

> Two instances have two class locks.

❌ Wrong. Static synchronization uses the class object's monitor.

### Mistake 4

> Object lock blocks class lock.

❌ Not automatically. They are different monitors.

### Mistake 5

> Class lock blocks every synchronized instance method.

❌ Not automatically.

### Mistake 6

> `synchronized(MyClass.class)` and static synchronized method use unrelated locks.

❌ They use the same class monitor for that class.

### Mistake 7

> Making a method static automatically makes it thread-safe.

❌ Static and synchronization are separate concepts.

---

# 25. Interview Questions

### Q1. What is an object lock?

The intrinsic monitor associated with a specific object instance.

### Q2. What is a class lock?

The intrinsic monitor associated with the corresponding `Class` object.

### Q3. What does an instance synchronized method lock?

`this`, the current object.

### Q4. What does a static synchronized method lock?

The class object's monitor.

### Q5. Can two different objects execute an instance synchronized method simultaneously?

Yes, because they have different object monitors.

### Q6. Can two different objects execute the same static synchronized method simultaneously?

No, because both calls use the same class monitor.

### Q7. Does an object lock block a class lock?

No, not by itself. They are different monitors.

### Q8. Is `synchronized(MyClass.class)` equivalent to a static synchronized method's lock?

Yes, for the same class monitor.

### Q9. Can an instance synchronized method and static synchronized method run concurrently?

Yes, because they normally acquire different monitors.

### Q10. Why is this distinction important?

Because correct thread-safety design depends on synchronizing the shared state with the correct common monitor.

### Q11. What happens if two references point to the same object and call synchronized instance methods?

They compete for the same object monitor.

### Q12. Why might class-level locking be dangerous for performance?

Because it can serialize unrelated operations across all callers using that class monitor.

---

# 26. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"In Java, an object lock is the intrinsic monitor of a particular object instance, while a class lock is the monitor of the corresponding Class object. An instance synchronized method uses the object's monitor, effectively `synchronized(this)`. A static synchronized method uses the class monitor, effectively `synchronized(MyClass.class)`. This means two different instances can execute an instance synchronized method concurrently because they have different monitors, but calls to a static synchronized method from different instances still compete for the same class monitor. Also, an instance lock and a class lock are independent, so holding one does not automatically block the other. The key interview point is that synchronization is always about a specific monitor: to get mutual exclusion, all competing threads must synchronize on the same lock."**

---

# 27. Quick Revision ⭐⭐⭐⭐⭐

```text
Object Lock
    ↓
Specific instance
    ↓
instance synchronized method
    ↓
this monitor
```

```text
Class Lock
    ↓
Class object
    ↓
static synchronized method
    ↓
MyClass.class monitor
```

### Remember

```text
obj1 != obj2
→ different object monitors

MyClass.class
→ one class monitor for that class-loader context

instance synchronized
→ object lock

static synchronized
→ class lock

object lock ≠ class lock
```

### One-line memory trick

> **Object lock protects an instance; class lock protects class-level shared state.**

---

# 28. Practice Checklist

- [x] Object lock basics
- [x] Class lock basics
- [x] Same object vs different object
- [x] Multiple instance behavior
- [x] Static synchronized behavior
- [x] Instance vs class monitor
- [x] Explicit object lock
- [x] Explicit class lock
- [x] Mixed synchronization scenarios
- [x] Interview scenarios
- [x] Practice code
- [x] Common mistakes
- [x] Interview Q&A
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.15 — Intrinsic Monitor / Object Lock](../15-Intrinsic-Monitor-Object-Lock/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.17 — Reentrancy of `synchronized`**