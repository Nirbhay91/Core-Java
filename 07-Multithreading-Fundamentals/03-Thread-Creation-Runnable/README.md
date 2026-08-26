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

# 17. Complete End-to-End Real-World Example — Order Processing System ⭐⭐⭐⭐⭐

### 🎯 Scenario

Imagine an e-commerce application receives orders. After an order is created, a background task needs to perform a sequence of independent operations:

```text
Customer places order
        ↓
Order created
        ↓
Runnable task submitted to a Thread
        ↓
Validate order
        ↓
Reserve inventory
        ↓
Generate invoice
        ↓
Send confirmation
        ↓
Task completed
```

For this chapter we intentionally use **`Thread + Runnable` directly** so that you can clearly see how `Runnable` represents the work. In production, the same task would normally move to an `ExecutorService`/thread pool rather than creating a new raw thread per order.

### Complete code — every important line commented

```java
// Main class containing the complete runnable example.
public class OrderProcessingRunnableExample {

    // Simple domain object representing an order.
    static class Order {

        // Unique identifier of the order.
        private final String orderId;

        // Customer name associated with the order.
        private final String customerName;

        // Total amount of the order.
        private final double amount;

        // Constructor used to create an Order object.
        Order(String orderId, String customerName, double amount) {
            // Store the supplied order id in the instance field.
            this.orderId = orderId;

            // Store the supplied customer name in the instance field.
            this.customerName = customerName;

            // Store the supplied amount in the instance field.
            this.amount = amount;
        }

        // Getter used to read the order id.
        public String getOrderId() {
            // Return the order id.
            return orderId;
        }

        // Getter used to read the customer name.
        public String getCustomerName() {
            // Return the customer name.
            return customerName;
        }

        // Getter used to read the order amount.
        public double getAmount() {
            // Return the order amount.
            return amount;
        }
    }

    // Runnable represents the background business task.
    static class OrderProcessingTask implements Runnable {

        // Store the order that this task needs to process.
        private final Order order;

        // Constructor receives the order to process.
        OrderProcessingTask(Order order) {
            // Save the order reference in the task object.
            this.order = order;
        }

        // Thread executes this method when start() is called.
        @Override
        public void run() {

            // Read the name of the thread currently executing this task.
            String threadName = Thread.currentThread().getName();

            // Print a message showing that processing has started.
            System.out.println(
                    "[" + threadName + "] Started order processing: "
                            + order.getOrderId()
            );

            // Validate the incoming order before processing it further.
            if (!validateOrder()) {
                // Print an error message when validation fails.
                System.out.println(
                        "[" + threadName + "] Order validation failed: "
                                + order.getOrderId()
                );

                // Stop the task because the order is invalid.
                return;
            }

            // Reserve the inventory required for this order.
            reserveInventory();

            // Generate an invoice for the customer.
            generateInvoice();

            // Send a confirmation message to the customer.
            sendConfirmation();

            // Print a success message after every step is completed.
            System.out.println(
                    "[" + threadName + "] Completed order processing: "
                            + order.getOrderId()
            );
        }

        // Method representing order validation business logic.
        private boolean validateOrder() {

            // Print which order is currently being validated.
            System.out.println(
                    "[" + Thread.currentThread().getName()
                            + "] Validating order: " + order.getOrderId()
            );

            // Simulate a small amount of processing time.
            sleep(500);

            // Return true when the order has a positive amount.
            return order.getAmount() > 0;
        }

        // Method representing inventory reservation logic.
        private void reserveInventory() {

            // Print an inventory reservation message.
            System.out.println(
                    "[" + Thread.currentThread().getName()
                            + "] Reserving inventory for: " + order.getOrderId()
            );

            // Simulate inventory service processing time.
            sleep(700);
        }

        // Method representing invoice-generation logic.
        private void generateInvoice() {

            // Print an invoice-generation message.
            System.out.println(
                    "[" + Thread.currentThread().getName()
                            + "] Generating invoice for: " + order.getOrderId()
            );

            // Simulate invoice processing time.
            sleep(400);

            // Print the invoice amount.
            System.out.println(
                    "[" + Thread.currentThread().getName()
                            + "] Invoice amount: " + order.getAmount()
            );
        }

        // Method representing customer notification logic.
        private void sendConfirmation() {

            // Print a confirmation message.
            System.out.println(
                    "[" + Thread.currentThread().getName()
                            + "] Sending confirmation to: "
                            + order.getCustomerName()
            );

            // Simulate notification service processing time.
            sleep(300);
        }

        // Utility method used to pause the current thread safely.
        private void sleep(long milliseconds) {

            // Start a try block because Thread.sleep() can throw InterruptedException.
            try {

                // Pause the currently executing thread for the requested duration.
                Thread.sleep(milliseconds);

            } catch (InterruptedException e) {

                // Restore the interrupted status of the current thread.
                Thread.currentThread().interrupt();

                // Print a message explaining that processing was interrupted.
                System.out.println(
                        "[" + Thread.currentThread().getName()
                                + "] Task interrupted for order: "
                                + order.getOrderId()
                );
            }
        }
    }

    // Main method is the application entry point.
    public static void main(String[] args) throws InterruptedException {

        // Create the first order.
        Order order1 = new Order(
                "ORD-1001",
                "Nirbhay",
                2499.00
        );

        // Create the second order.
        Order order2 = new Order(
                "ORD-1002",
                "Rahul",
                1599.00
        );

        // Create a Runnable task for the first order.
        Runnable task1 = new OrderProcessingTask(order1);

        // Create a Runnable task for the second order.
        Runnable task2 = new OrderProcessingTask(order2);

        // Create a thread and associate the first Runnable task with it.
        Thread worker1 = new Thread(task1, "order-worker-1");

        // Create another thread and associate the second Runnable task with it.
        Thread worker2 = new Thread(task2, "order-worker-2");

        // Print a message before starting background processing.
        System.out.println("[main] Starting order processing...");

        // Start the first worker thread.
        worker1.start();

        // Start the second worker thread.
        worker2.start();

        // Wait until the first worker thread finishes.
        worker1.join();

        // Wait until the second worker thread finishes.
        worker2.join();

        // Print a final message after both worker threads have completed.
        System.out.println("[main] All orders processed successfully.");
    }
}
```

### 🔍 What this example teaches

```text
Order
   ↓
Domain data

OrderProcessingTask implements Runnable
   ↓
Business task / work

Thread
   ↓
Execution mechanism

worker1.start()
worker2.start()
   ↓
run() executes concurrently
```

The most important design separation is:

```text
OrderProcessingTask
        ↓
WHAT work should happen?

Thread
        ↓
HOW/WHERE that work executes?
```

This directly demonstrates why `Runnable` is a task abstraction. fileciteturn482file0

---

## 18. Why This Real-World Example Is Better Than a Simple Print Statement

A simple example:

```java
new Thread(() -> System.out.println("Hello")).start();
```

shows syntax, but the order-processing example demonstrates the interview-level design:

```text
Domain object
   ↓
Task object
   ↓
Runnable
   ↓
Thread
   ↓
start()
   ↓
Concurrent execution
```

It also demonstrates:

- thread naming
- multiple tasks
- `join()`
- checked interruption handling
- task/business separation
- lifecycle ownership by `Thread`
- why the task itself does not create the thread

---

## 19. Real-World Design Improvement — Move to ExecutorService ⭐⭐⭐⭐⭐

The previous example intentionally creates raw threads because the purpose is to learn `Runnable`.

For production systems, this:

```java
Thread worker = new Thread(task);
worker.start();
```

would commonly become:

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(task);
executor.shutdown();
```

The separation becomes:

```text
Runnable
   ↓
Business task
   ↓
ExecutorService
   ↓
Thread Pool
   ↓
Worker Thread
```

This is exactly why learning `Runnable` first is important before moving to `ExecutorService`.

---

## 20. Interview Scenario — Why Not Create One Thread Per Order?

Suppose the system receives:

```text
10 orders
100 orders
10,000 orders
100,000 orders
```

Creating one new `Thread` for every order does not scale well.

The learning example is intentionally:

```text
Order → Runnable → Thread
```

The production pattern is usually:

```text
Order → Runnable → ExecutorService → Thread Pool
```

### 5-Year Interview Answer

> **"I would use Runnable to model the order-processing task, but I would not create a raw Thread for every production order. For a real application I would submit these Runnable tasks to an ExecutorService with an appropriately sized thread pool. That gives me controlled concurrency, thread reuse, queueing and lifecycle management."**

---

## 21. Important `join()` Point

In the example:

```java
worker1.join();
worker2.join();
```

the `main` thread waits for both worker threads to finish.

```text
main
 │
 ├── worker1.start()
 │
 ├── worker2.start()
 │
 ├── worker1.join() ── wait
 │
 └── worker2.join() ── wait
          ↓
     workers finished
          ↓
      main continues
```

`join()` does not make `Runnable` asynchronous; it is a coordination mechanism for the `Thread` objects.

---

## 22. Runnable Thread-Safety Follow-up

Our `OrderProcessingTask` contains:

```java
private final Order order;
```

The task does not mutate shared state between worker threads.

That makes the example easier to reason about.

But this would be dangerous:

```java
class OrderProcessingTask implements Runnable {
    private int processedOrders;

    @Override
    public void run() {
        processedOrders++;
    }
}
```

If the same task object is executed by multiple threads, `processedOrders++` is a read-modify-write operation and is not automatically atomic.

Possible solutions depend on the design:

```text
AtomicInteger
synchronized
Lock
thread confinement
immutable state
executor-level aggregation
```

---

## 23. Common Mistakes ⭐⭐⭐⭐⭐

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

### Mistake 6 — Creating unlimited raw threads in production

```java
for (...) {
    new Thread(task).start();
}
```

❌ Thread creation has overhead and uncontrolled concurrency can exhaust resources.

Prefer an executor/thread pool for production workloads.

---

## 24. Interview Questions

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

### Q11. Why is `join()` used in a Runnable + Thread example?

`join()` lets the calling thread wait for a worker thread to terminate before continuing.

### Q12. Why is a thread pool better than creating one Thread per request?

It reuses worker threads, limits concurrency, manages task queueing and gives better control over resources.

---

## 25. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"Runnable is a functional interface used to represent a task independently of the thread that executes it. We implement `run()` and pass the Runnable to a `Thread`, then call `start()`. Runnable itself does not create a thread. The major advantage over extending Thread is separation of concerns: Runnable represents the work, while Thread represents the execution mechanism. It also avoids Java's single-class-inheritance limitation and works naturally with lambdas and executor frameworks. However, Runnable does not make shared state thread-safe; synchronization is still required when multiple threads access mutable shared data. In a real application I would model work as Runnable but usually submit it to an ExecutorService/thread pool instead of creating a raw Thread for every request. For tasks that need a return value, Callable is generally used instead." 

---

## 26. Quick Revision

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

Learning/demo:
Runnable → Thread

Production:
Runnable → ExecutorService → Thread Pool
```

---

## 27. Completion Checklist

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
- [x] **End-to-end real-world order processing example**
- [x] **Every important line commented in real-world example**
- [x] `join()` coordination
- [x] Production design improvement
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