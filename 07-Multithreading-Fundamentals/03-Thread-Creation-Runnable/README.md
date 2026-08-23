# 7.3 — Thread Creation using `Runnable`

## 🎯 Objective

Understand `Runnable` as a task abstraction, how to execute it using `Thread`, why it is generally preferred over extending `Thread`, and the interview-level differences between `Thread` and `Runnable`.

---

## 1. What is `Runnable`?

`Runnable` is a functional interface from `java.lang` that represents a task which can be executed.

```java
@FunctionalInterface
public interface Runnable {
    void run();
}
```

The key idea is:

```text
Runnable → What work should be done?
Thread   → Which execution mechanism runs that work?
```

This separates the **task** from the **thread**.

---

## 2. Basic Example

```java
class Worker implements Runnable {

    @Override
    public void run() {
        System.out.println("Task executed by: "
                + Thread.currentThread().getName());
    }
}

public class Main {

    public static void main(String[] args) {
        Runnable task = new Worker();
        Thread thread = new Thread(task);

        thread.start();
    }
}
```

Flow:

```text
Worker implements Runnable
        ↓
Runnable object represents task
        ↓
Thread receives Runnable
        ↓
start()
        ↓
Thread executes Runnable.run()
```

---

## 3. Why Use `Runnable`?

The main advantage is **separation of responsibility**.

```text
Runnable
   ↓
Task / business work

Thread
   ↓
Execution mechanism
```

With `Thread` inheritance, these concerns are combined:

```text
class Worker extends Thread
       ↓
Thread + Task together
```

With `Runnable`:

```text
class Worker implements Runnable
       ↓
Task only

new Thread(worker)
       ↓
Execution mechanism
```

---

## 4. `Runnable` and Single Inheritance ⭐⭐⭐⭐⭐

Java allows a class to extend only one class.

Suppose:

```java
class PaymentService extends BaseService {
    // business logic
}
```

This class cannot also do:

```java
class PaymentService extends BaseService, Thread { // invalid
}
```

But it can implement `Runnable`:

```java
class PaymentService extends BaseService implements Runnable {

    @Override
    public void run() {
        // task
    }
}
```

This is one of the strongest interview reasons for preferring `Runnable` over extending `Thread`.

---

## 5. Runnable with Lambda

Because `Runnable` has exactly one abstract method, it is a **functional interface**.

Therefore it can be implemented using a lambda:

```java
Runnable task = () -> {
    System.out.println("Task is running");
};

Thread thread = new Thread(task);
thread.start();
```

Even shorter:

```java
new Thread(() -> {
    System.out.println("Task is running");
}).start();
```

---

## 6. `Runnable` vs `Thread` ⭐⭐⭐⭐⭐

| Feature | `Thread` | `Runnable` |
|---|---|---|
| Represents | Thread/execution object | Task to execute |
| Type | Class | Functional interface |
| Inheritance impact | Consumes class inheritance | Does not consume class inheritance |
| Task separation | Lower | Better |
| Lambda support | Not directly as a Thread subclass | Yes |
| Reusability as a task | Less flexible | Better |
| Works with executors | Possible, but not the preferred abstraction | Natural fit |
| Recommended design | Mainly for learning/specialized cases | Preferred for task abstraction |

### Interview one-liner

> `Runnable` represents the task, while `Thread` represents an execution mechanism that can run that task.

---

## 7. Important: `Runnable` Does Not Create a Thread

This is a common interview trap.

```java
Runnable task = () -> {
    System.out.println(Thread.currentThread().getName());
};
```

Creating the `Runnable` object does **not** create a new thread.

```text
Runnable object
      ↓
Only represents task
```

A thread is created when you create a `Thread` and start it:

```java
Thread thread = new Thread(task);
thread.start();
```

---

## 8. `run()` vs `start()` with Runnable

### Correct

```java
Runnable task = () -> {
    System.out.println(Thread.currentThread().getName());
};

Thread thread = new Thread(task);
thread.start();
```

### Direct `run()` call

```java
thread.run();
```

This does **not** create a new thread. It executes `run()` as a normal method call on the current thread.

```text
main
  ↓
thread.run()
  ↓
Runnable.run()
```

### Remember

```text
start() → new thread execution
run()   → normal method invocation when called directly
```

---

## 9. Thread Naming with Runnable

```java
Runnable task = () -> {
    System.out.println(
            "Running on: "
                    + Thread.currentThread().getName()
    );
};

Thread thread = new Thread(task, "payment-worker");
thread.start();
```

Naming threads is useful for:

- Debugging
- Logging
- Monitoring
- Production troubleshooting

---

## 10. Multiple Threads Using the Same Runnable

A single `Runnable` object can be supplied to multiple `Thread` objects.

```java
Runnable task = () -> {
    System.out.println(
            "Running: "
                    + Thread.currentThread().getName()
    );
};

Thread t1 = new Thread(task, "worker-1");
Thread t2 = new Thread(task, "worker-2");

 t1.start();
 t2.start();
```

The task object can therefore be separated from the thread instances.

> **Important:** Reusing the same `Runnable` object does not make its mutable state thread-safe. If the `Runnable` contains shared mutable fields, synchronization or another thread-safety strategy may still be required.

---

## 11. Runnable with Shared State

Example:

```java
class CounterTask implements Runnable {

    private int count = 0;

    @Override
    public void run() {
        count++;
        System.out.println(count);
    }
}
```

If multiple threads use the same `CounterTask` instance:

```java
CounterTask task = new CounterTask();

new Thread(task).start();
new Thread(task).start();
```

then `count` is shared between those threads.

This can introduce race conditions.

The important distinction is:

```text
Runnable ≠ automatically thread-safe
```

`Runnable` only represents a task.

---

## 12. Runnable with Anonymous Class

Before lambdas, an anonymous class was commonly used:

```java
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Task running");
    }
};

new Thread(task).start();
```

Equivalent lambda:

```java
Runnable task = () ->
        System.out.println("Task running");
```

---

## 13. Runnable vs Callable — Important Preview

`Runnable` does not return a result and its `run()` method cannot declare checked exceptions.

```java
Runnable task = () -> {
    // no return value
};
```

`Callable<V>` is designed for tasks that return a result and can throw checked exceptions.

```text
Runnable
  ↓
No result

Callable<V>
  ↓
Returns V
  ↓
Can throw Exception
```

`Callable` and `Future` will be covered later in the Executor Framework section.

---

## 14. Runnable with `Thread` Constructor

Common constructor form:

```java
Thread thread = new Thread(runnable);
```

With a name:

```java
Thread thread = new Thread(runnable, "worker-1");
```

The important relationship is:

```text
Thread
  │
  └── target Runnable
          │
          └── run()
```

---

## 15. Runnable and Thread Lifecycle

The lifecycle belongs to the `Thread` object, not to the `Runnable` task.

```text
Runnable
  ↓
Task definition

Thread
  ↓
NEW
  ↓ start()
RUNNABLE
  ↓
Execution
  ↓
TERMINATED
```

A `Runnable` itself does not have `Thread.State`.

This is an important distinction.

---

## 16. Why Runnable Fits Executor Framework

The task/execution separation becomes even more valuable when using thread pools.

Instead of:

```java
new Thread(task).start();
```

an executor can manage execution:

```java
ExecutorService executor = ...;
executor.submit(task);
```

Conceptually:

```text
Runnable Task
      ↓
ExecutorService
      ↓
Thread Pool
      ↓
Worker Thread
      ↓
Task executes
```

This avoids manually creating a new thread for every task and is generally more appropriate for production applications.

---

## 17. Common Mistakes ⭐⭐⭐⭐⭐

### Mistake 1 — Thinking Runnable creates a thread

```java
Runnable r = ...;
```

❌ No thread has started.

### Mistake 2 — Calling `run()` instead of `start()`

```java
new Thread(r).run();
```

❌ Executes synchronously on the caller thread.

### Mistake 3 — Assuming Runnable is thread-safe

```java
class Task implements Runnable {
    int counter;
}
```

❌ `Runnable` does not automatically synchronize shared mutable state.

### Mistake 4 — Assuming the same Runnable means the same thread

```java
new Thread(task).start();
new Thread(task).start();
```

These are two separate thread instances executing the same task object.

### Mistake 5 — Assuming execution order

```java
t1.start();
t2.start();
```

❌ Does not guarantee that `t1` runs before `t2`.

---

## 18. Interview Questions

### Q1. What is Runnable?

`Runnable` is a functional interface representing a task that can be executed by a thread or an executor.

### Q2. Does Runnable create a thread?

No. It represents the task. A `Thread` or executor is needed to execute it.

### Q3. Why prefer Runnable over extending Thread?

It separates task from execution and preserves the class's ability to extend another class.

### Q4. Can a class extend another class and implement Runnable?

Yes.

```java
class PaymentService extends BaseService implements Runnable {
    public void run() { }
}
```

### Q5. Can the same Runnable be used by multiple threads?

Yes. But shared mutable state inside the Runnable must be made thread-safe.

### Q6. What happens when `run()` is called directly?

It executes synchronously on the calling thread; no new thread is created.

### Q7. Why is Runnable a functional interface?

Because it has exactly one abstract method: `run()`.

### Q8. Can Runnable return a value?

No. `run()` returns `void`. Use `Callable<V>` when a task needs to return a result.

### Q9. Can Runnable throw checked exceptions from `run()`?

The `run()` method does not declare checked exceptions, so a Runnable implementation cannot declare arbitrary checked exceptions from `run()`.

### Q10. How does Runnable work with ExecutorService?

The Runnable represents the task and the executor manages when and on which worker thread the task executes.

---

## 19. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Runnable is a functional interface used to represent a task independently of the thread that executes it. We implement `run()` and pass the Runnable to a `Thread`, then call `start()`. Runnable itself does not create a thread. The major advantage over extending Thread is separation of concerns: Runnable represents the work, while Thread represents the execution mechanism. It also avoids Java's single-class-inheritance limitation and works naturally with lambdas and executor frameworks. However, Runnable does not make shared state thread-safe; synchronization is still required when multiple threads access mutable shared data. For tasks that need a return value, `Callable` is generally used instead." 

---

## 20. Quick Revision

```text
Runnable
   ↓
Task abstraction
   ↓
Implement run()
   ↓
Pass to Thread
   ↓
thread.start()
   ↓
New thread executes run()
```

### Core Difference

```text
Thread   = execution mechanism
Runnable = task
```

### Remember

```text
Runnable object ≠ Thread

run() directly → current thread
start()        → new thread execution

Runnable → no return value
Callable → returns value
```

---

## 21. Completion Checklist

- [x] `Runnable` definition
- [x] Functional interface
- [x] Basic implementation
- [x] `Runnable` vs `Thread`
- [x] Single inheritance advantage
- [x] Lambda implementation
- [x] `Runnable` does not create a thread
- [x] `start()` vs `run()`
- [x] Multiple threads with one Runnable
- [x] Shared mutable state warning
- [x] Anonymous class
- [x] Runnable vs Callable preview
- [x] Thread lifecycle relationship
- [x] Executor Framework connection
- [x] Common mistakes
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← 7.2 — Thread Creation using `Thread` class](../02-Thread-Creation-Thread-Class/README.md)

[🏠 Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.4 — Thread Lifecycle**