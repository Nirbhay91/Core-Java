# 8.26 — `BlockingQueue` Implementations

> **Goal:** Understand `BlockingQueue`, producer-consumer coordination, blocking vs non-blocking queue operations, interruption, bounded queues, backpressure, and the major `BlockingQueue` implementations.

---

## 1. What is `BlockingQueue`? ⭐⭐⭐⭐⭐

`BlockingQueue` is a thread-safe queue from `java.util.concurrent` designed for concurrent producer-consumer workflows.

Its key feature is **blocking behavior**:

```text
Producer
   ↓
put()
   ↓
Queue full?
   ├── Yes → wait until space is available
   └── No  → insert

Consumer
   ↓
take()
   ↓
Queue empty?
   ├── Yes → wait until an element is available
   └── No  → remove
```

This lets producers and consumers coordinate without manually writing `wait()` / `notify()` logic.

---

# 2. Package and Interface ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.BlockingQueue;
```

`BlockingQueue<E>` extends `Queue<E>`.

Common implementations include:

```text
ArrayBlockingQueue
LinkedBlockingQueue
PriorityBlockingQueue
DelayQueue
SynchronousQueue
LinkedTransferQueue
```

---

# 3. The Four Operation Families ⭐⭐⭐⭐⭐

This is one of the most important interview tables.

| Operation | When queue is full/empty |
|---|---|
| `add(e)` | throws exception |
| `offer(e)` | returns `false` |
| `put(e)` | blocks |
| `offer(e, timeout, unit)` | waits up to timeout |
| `remove()` | throws exception |
| `poll()` | returns `null` |
| `take()` | blocks |
| `poll(timeout, unit)` | waits up to timeout |
| `element()` | throws exception if empty |
| `peek()` | returns `null` if empty |

### Golden memory trick

```text
add     → Exception
offer   → false
put     → Wait

remove  → Exception
poll    → null
 take   → Wait
```

---

# 4. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BasicBlockingQueue {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> queue =
                new ArrayBlockingQueue<>(3);

        queue.put("Java");
        queue.put("Spring");

        System.out.println(queue.take());
        System.out.println(queue.take());
    }
}
```

---

# 5. Why BlockingQueue Exists ⭐⭐⭐⭐⭐

Without `BlockingQueue`, a producer-consumer implementation often needs:

```text
synchronized
wait()
notify()/notifyAll()
condition checks
spurious-wakeup-safe loops
manual queue management
```

With `BlockingQueue`:

```java
queue.put(item);
queue.take();
```

The queue handles the coordination mechanism.

---

# 6. Producer-Consumer Pattern ⭐⭐⭐⭐⭐

The classic architecture:

```text
Producer 1 ─┐
Producer 2 ─┼──→ BlockingQueue ──→ Consumer 1
Producer 3 ─┘                       Consumer 2
```

The queue acts as the thread-safe handoff point.

---

# 7. Practice — Basic Producer Consumer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerDemo {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(3);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    queue.put(i);
                    System.out.println("Produced: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    int value = queue.take();
                    System.out.println("Consumed: " + value);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }
}
```

---

# 8. `put()` ⭐⭐⭐⭐⭐

`put()` inserts an element, waiting if necessary until space becomes available.

```java
queue.put("Java");
```

It throws:

```java
InterruptedException
```

if the waiting thread is interrupted.

### Important

`put()` is a **blocking** operation.

---

# 9. Practice — `put()` Blocks ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class PutBlocks {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(1);

        queue.put(100);

        Thread producer = new Thread(() -> {
            try {
                System.out.println("Trying to put 200...");
                queue.put(200);
                System.out.println("Put completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();

        Thread.sleep(1000);

        System.out.println("Consumer takes: " + queue.take());

        producer.join();
    }
}
```

The producer waits because the bounded queue is full.

---

# 10. `take()` ⭐⭐⭐⭐⭐

`take()` removes and returns the head element.

If the queue is empty:

```java
queue.take();
```

blocks until an element becomes available.

It is interruptible and therefore declares `InterruptedException`.

---

# 11. Practice — `take()` Blocks ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class TakeBlocks {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<String> queue =
                new ArrayBlockingQueue<>(2);

        Thread consumer = new Thread(() -> {
            try {
                System.out.println("Waiting for data...");
                String value = queue.take();
                System.out.println("Received: " + value);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        consumer.start();

        Thread.sleep(1000);
        queue.put("Java");

        consumer.join();
    }
}
```

---

# 12. `offer()` ⭐⭐⭐⭐⭐

`offer()` attempts insertion without waiting indefinitely.

```java
boolean added = queue.offer("Java");
```

If the queue is full:

```text
false
```

is returned.

---

# 13. Practice — `offer()` ⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class OfferDemo {

    public static void main(String[] args) {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(1);

        System.out.println(queue.offer(10));
        System.out.println(queue.offer(20));
        System.out.println(queue);
    }
}
```

Expected:

```text
true
false
[10]
```

---

# 14. Timed `offer()` ⭐⭐⭐⭐⭐

```java
queue.offer(value, 2, TimeUnit.SECONDS);
```

This waits up to the specified duration for capacity.

```java
boolean added = queue.offer(
        "Java",
        2,
        TimeUnit.SECONDS
);
```

It returns `false` if space does not become available in time.

---

# 15. Practice — Timed `offer()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

public class TimedOfferDemo {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(1);

        queue.put(10);

        long start = System.currentTimeMillis();

        boolean result = queue.offer(
                20,
                2,
                TimeUnit.SECONDS
        );

        long elapsed = System.currentTimeMillis() - start;

        System.out.println("Added = " + result);
        System.out.println("Waited approximately = " + elapsed + " ms");
    }
}
```

---

# 16. `poll()` ⭐⭐⭐⭐⭐

`poll()` removes the head if one is immediately available.

If empty:

```java
null
```

is returned.

```java
String value = queue.poll();
```

It does not wait indefinitely.

---

# 17. Timed `poll()` ⭐⭐⭐⭐⭐

```java
queue.poll(2, TimeUnit.SECONDS);
```

This waits up to the specified duration for an element.

If none arrives within the timeout:

```text
null
```

is returned.

---

# 18. Practice — Timed `poll()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

public class TimedPollDemo {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<String> queue =
                new ArrayBlockingQueue<>(2);

        long start = System.currentTimeMillis();

        String value = queue.poll(
                2,
                TimeUnit.SECONDS
        );

        long elapsed = System.currentTimeMillis() - start;

        System.out.println("Value = " + value);
        System.out.println("Waited approximately = " + elapsed + " ms");
    }
}
```

---

# 19. `add()` vs `offer()` vs `put()` ⭐⭐⭐⭐⭐

```text
Queue capacity = 2
Current size = 2
```

### `add()`

```java
queue.add(3);
```

→ `IllegalStateException`

### `offer()`

```java
queue.offer(3);
```

→ `false`

### `put()`

```java
queue.put(3);
```

→ waits until space exists.

---

# 20. Practice — Compare Insert Operations ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class InsertOperations {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(1);

        queue.add(1);

        try {
            queue.add(2);
        } catch (IllegalStateException e) {
            System.out.println("add(): exception");
        }

        System.out.println("offer(): " + queue.offer(2));

        Thread consumer = new Thread(() -> {
            try {
                Thread.sleep(500);
                queue.take();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        consumer.start();

        queue.put(2);
        System.out.println("put(): completed");

        consumer.join();
    }
}
```

---

# 21. `remove()` vs `poll()` vs `take()` ⭐⭐⭐⭐⭐

If the queue is empty:

```text
remove() → exception
poll()   → null
take()   → blocks
```

This is another must-memorize interview table.

---

# 22. Bounded vs Unbounded Queue ⭐⭐⭐⭐⭐

A bounded queue has a fixed capacity.

Example:

```java
new ArrayBlockingQueue<>(100);
```

An effectively unbounded queue can grow until memory/resource limits are reached.

Example:

```java
new LinkedBlockingQueue<>();
```

### Important production point

Unbounded queues can hide overload because producers continue enqueueing work instead of applying immediate backpressure.

---

# 23. Backpressure ⭐⭐⭐⭐⭐

Suppose:

```text
Producer = 10,000 requests/sec
Consumer = 2,000 requests/sec
```

The queue will grow if there is no effective limit.

A bounded `BlockingQueue` can apply backpressure:

```text
Queue full
   ↓
Producer blocks / rejects / times out
   ↓
System avoids unlimited queue growth
```

This is extremely important in production systems.

---

# 24. Practice — Bounded Queue as Backpressure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class BackpressureDemo {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(2);

        Thread slowConsumer = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    Thread.sleep(1000);
                    System.out.println("Consumed: " + queue.take());
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    System.out.println("Producing: " + i);
                    queue.put(i);
                    System.out.println("Queued: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        slowConsumer.start();
        producer.start();

        producer.join();
        slowConsumer.interrupt();
        slowConsumer.join();
    }
}
```

Observe how the producer eventually waits for the slow consumer to create capacity.

---

# 25. `ArrayBlockingQueue` ⭐⭐⭐⭐⭐

Characteristics:

```text
bounded
array-backed
FIFO
optional fairness
fixed capacity
```

Example:

```java
BlockingQueue<String> queue =
        new ArrayBlockingQueue<>(100);
```

It is a strong choice when you explicitly want a bounded FIFO buffer.

---

# 26. Fairness in `ArrayBlockingQueue` ⭐⭐⭐⭐

You can choose fairness:

```java
new ArrayBlockingQueue<>(10, true);
```

The fairness setting affects waiting thread scheduling order.

Do not assume fairness is free: stronger fairness guarantees can reduce throughput.

---

# 27. Practice — Fair `ArrayBlockingQueue` ⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class FairQueueDemo {

    public static void main(String[] args) {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(2, true);

        queue.add(1);
        queue.add(2);

        System.out.println(queue);
    }
}
```

---

# 28. `LinkedBlockingQueue` ⭐⭐⭐⭐⭐

Characteristics:

```text
linked-node based
FIFO
optionally bounded
```

Example with explicit capacity:

```java
BlockingQueue<String> queue =
        new LinkedBlockingQueue<>(100);
```

You can also construct it without an explicit capacity, but that creates a queue with a very large maximum capacity (`Integer.MAX_VALUE`), which should not be confused with having infinite memory.

---

# 29. Why Explicit Capacity Is Often Better ⭐⭐⭐⭐⭐

Prefer:

```java
new LinkedBlockingQueue<>(1000);
```

over casually using:

```java
new LinkedBlockingQueue<>();
```

when workload control matters.

A bounded queue makes overload visible and allows backpressure.

---

# 30. `PriorityBlockingQueue` ⭐⭐⭐⭐⭐

`PriorityBlockingQueue` is:

```text
thread-safe
blocking when empty on take()
priority ordered
unbounded in its standard capacity model
```

It does **not** provide FIFO ordering among elements with equal priority unless the element comparator/model explicitly establishes such ordering.

Example:

```java
BlockingQueue<Integer> queue =
        new PriorityBlockingQueue<>();

queue.put(30);
queue.put(10);
queue.put(20);

System.out.println(queue.take()); // 10
```

---

# 31. Practice — Priority Blocking Queue ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityQueueDemo {

    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<Integer> queue =
                new PriorityBlockingQueue<>();

        queue.put(50);
        queue.put(10);
        queue.put(30);

        System.out.println(queue.take());
        System.out.println(queue.take());
        System.out.println(queue.take());
    }
}
```

Expected order:

```text
10
30
50
```

---

# 32. `DelayQueue` ⭐⭐⭐⭐⭐

`DelayQueue` contains elements that become available only after their delay expires.

Elements must implement `Delayed`.

Conceptually:

```text
Task added
   ↓
Delay active
   ↓
take() waits
   ↓
Delay expires
   ↓
Task becomes available
```

---

# 33. Practice — `DelayQueue` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Delayed;
import java.util.concurrent.DelayQueue;
import java.util.concurrent.TimeUnit;

public class DelayQueueDemo {

    static class DelayedTask implements Delayed {
        private final String name;
        private final long deadline;

        DelayedTask(String name, long delayMillis) {
            this.name = name;
            this.deadline = System.nanoTime()
                    + TimeUnit.MILLISECONDS.toNanos(delayMillis);
        }

        @Override
        public long getDelay(TimeUnit unit) {
            long remaining = deadline - System.nanoTime();
            return unit.convert(remaining, TimeUnit.NANOSECONDS);
        }

        @Override
        public int compareTo(Delayed other) {
            return Long.compare(
                    getDelay(TimeUnit.NANOSECONDS),
                    other.getDelay(TimeUnit.NANOSECONDS)
            );
        }

        @Override
        public String toString() {
            return name;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        DelayQueue<DelayedTask> queue = new DelayQueue<>();

        queue.put(new DelayedTask("Task-A", 2000));
        queue.put(new DelayedTask("Task-B", 500));

        System.out.println("Taking: " + queue.take());
        System.out.println("Taking: " + queue.take());
    }
}
```

---

# 34. `SynchronousQueue` ⭐⭐⭐⭐⭐

`SynchronousQueue` has no normal internal storage capacity.

Each insertion waits for a corresponding removal.

Think:

```text
Producer ── handoff ──→ Consumer
```

It is useful for direct handoff designs and is used by some executor configurations.

---

# 35. Practice — Direct Handoff ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.SynchronousQueue;

public class SynchronousQueueDemo {

    public static void main(String[] args) throws InterruptedException {
        SynchronousQueue<String> queue =
                new SynchronousQueue<>();

        Thread producer = new Thread(() -> {
            try {
                System.out.println("Producer putting...");
                queue.put("Java");
                System.out.println("Producer completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("Consumer got: " + queue.take());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }
}
```

---

# 36. `LinkedTransferQueue` ⭐⭐⭐⭐

`LinkedTransferQueue` supports transfer-style producer-consumer handoff.

Important method:

```java
transfer(element)
```

The producer can wait until a consumer receives the element.

It is useful when you need stronger handoff semantics than simply enqueueing an item.

---

# 37. Practice — `transfer()` ⭐⭐⭐⭐

```java
import java.util.concurrent.LinkedTransferQueue;

public class TransferQueueDemo {

    public static void main(String[] args) throws InterruptedException {
        LinkedTransferQueue<String> queue =
                new LinkedTransferQueue<>();

        Thread producer = new Thread(() -> {
            try {
                System.out.println("Transferring...");
                queue.transfer("Java");
                System.out.println("Transfer completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("Received: " + queue.take());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }
}
```

---

# 38. Interrupting a Blocked `put()` ⭐⭐⭐⭐⭐

Blocking operations such as `put()` are interruptible.

```java
try {
    queue.put(value);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Best practice

Usually restore the interrupt status:

```java
Thread.currentThread().interrupt();
```

unless the surrounding design intentionally consumes the interruption.

---

# 39. Practice — Interrupt Blocked Producer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class InterruptBlockedProducer {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(1);

        queue.put(1);

        Thread producer = new Thread(() -> {
            try {
                queue.put(2);
            } catch (InterruptedException e) {
                System.out.println("Producer interrupted");
                Thread.currentThread().interrupt();
            }
        });

        producer.start();

        Thread.sleep(500);
        producer.interrupt();

        producer.join();
    }
}
```

---

# 40. `remainingCapacity()` ⭐⭐⭐⭐

For bounded implementations, you can inspect available capacity:

```java
int remaining = queue.remainingCapacity();
```

Do not use this as a guarantee for a later operation in concurrent code.

Between checking and acting, another thread may change the queue.

### Trap

```java
if (queue.remainingCapacity() > 0) {
    queue.put(value);
}
```

This is not an atomic check-then-act workflow.

---

# 41. Practice — Check-Then-Act Trap ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class CapacityRace {

    public static void main(String[] args) {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(10);

        if (queue.remainingCapacity() > 0) {
            // Another thread can consume the capacity before this operation.
            queue.offer(100);
        }
    }
}
```

Prefer the actual queue operation and inspect its result rather than relying on a separate pre-check.

---

# 42. Null Elements ⭐⭐⭐⭐⭐

`BlockingQueue` implementations do not permit `null` elements.

Why?

Because `poll()` uses `null` to indicate that no element is currently available.

Therefore:

```java
queue.put(null);
```

is invalid and results in `NullPointerException`.

---

# 43. Practice — Null Rejection ⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class NullElementDemo {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<String> queue =
                new ArrayBlockingQueue<>(2);

        try {
            queue.put(null);
        } catch (NullPointerException e) {
            System.out.println("Null elements are not allowed");
        }
    }
}
```

---

# 44. Queue Ordering ⭐⭐⭐⭐⭐

Do not assume every `BlockingQueue` is FIFO.

```text
ArrayBlockingQueue       → FIFO
LinkedBlockingQueue      → FIFO
PriorityBlockingQueue    → priority order
DelayQueue                → delay availability/order
SynchronousQueue          → direct handoff
```

Always check the implementation's semantics.

---

# 45. `drainTo()` ⭐⭐⭐⭐

`drainTo()` transfers available elements into another collection.

```java
List<Integer> batch = new ArrayList<>();
queue.drainTo(batch, 100);
```

This can be useful for batch processing.

Important: `drainTo()` does not wait for elements to arrive; it drains what is currently available.

---

# 46. Practice — Batch Consumption ⭐⭐⭐⭐⭐

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;

public class DrainToDemo {

    public static void main(String[] args) {
        ArrayBlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(10);

        queue.add(1);
        queue.add(2);
        queue.add(3);
        queue.add(4);

        List<Integer> batch = new ArrayList<>();

        int drained = queue.drainTo(batch, 3);

        System.out.println("Drained = " + drained);
        System.out.println("Batch = " + batch);
        System.out.println("Remaining = " + queue);
    }
}
```

---

# 47. Multiple Producers and Consumers ⭐⭐⭐⭐⭐

`BlockingQueue` naturally supports multiple producers and consumers.

```text
Producer A ─┐
Producer B ─┼─→ Queue ─→ Consumer X
Producer C ─┘          └→ Consumer Y
```

No external synchronization should be added merely to make the queue operations themselves thread-safe.

---

# 48. Practice — Multiple Producers/Consumers ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class MultiProducerConsumer {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(5);

        Runnable producer = () -> {
            try {
                for (int i = 0; i < 5; i++) {
                    queue.put(i);
                    System.out.println(Thread.currentThread().getName()
                            + " produced " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        Runnable consumer = () -> {
            try {
                for (int i = 0; i < 5; i++) {
                    int value = queue.take();
                    System.out.println(Thread.currentThread().getName()
                            + " consumed " + value);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        Thread p1 = new Thread(producer, "Producer-1");
        Thread p2 = new Thread(producer, "Producer-2");
        Thread c1 = new Thread(consumer, "Consumer-1");
        Thread c2 = new Thread(consumer, "Consumer-2");

        p1.start();
        p2.start();
        c1.start();
        c2.start();

        p1.join();
        p2.join();
        c1.join();
        c2.join();
    }
}
```

---

# 49. `BlockingQueue` vs `wait/notify` ⭐⭐⭐⭐⭐

### Manual approach

```text
shared queue
+ synchronized
+ while(condition)
+ wait()
+ notifyAll()
+ interrupt handling
```

### BlockingQueue approach

```java
queue.put(item);
queue.take();
```

The abstraction removes a large amount of error-prone coordination code.

### Interview answer

> `BlockingQueue` is preferable for standard producer-consumer handoff because it encapsulates thread-safe queueing and blocking coordination instead of forcing application code to manage monitor ownership and wait/notify conditions manually.

---

# 50. `BlockingQueue` Does Not Mean "Everything Blocks" ⭐⭐⭐⭐⭐

The interface provides both blocking and non-blocking operations.

```text
Blocking:
put()
take()

Non-blocking:
offer()
poll()
peek()
```

Timed variants provide controlled waiting.

---

# 51. Production Scenario — Worker Pipeline ⭐⭐⭐⭐⭐

A common backend design:

```text
HTTP requests
     ↓
Task creation
     ↓
Bounded BlockingQueue
     ↓
Worker threads
     ↓
Database / external service
```

Benefits:

```text
bounded work-in-progress
backpressure
producer-consumer decoupling
controlled concurrency
```

---

# 52. Practice — Worker Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class WorkerPipeline {

    record Task(int id) {}

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Task> queue =
                new ArrayBlockingQueue<>(3);

        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    Task task = queue.take();
                    System.out.println("Processing task " + task.id());
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        for (int i = 1; i <= 10; i++) {
            queue.put(new Task(i));
        }

        Thread.sleep(500);
        worker.interrupt();
        worker.join();
    }
}
```

---

# 53. Choosing the Implementation ⭐⭐⭐⭐⭐

```text
Need bounded FIFO?
→ ArrayBlockingQueue

Need linked FIFO and configurable capacity?
→ LinkedBlockingQueue

Need priority ordering?
→ PriorityBlockingQueue

Need delayed availability?
→ DelayQueue

Need direct handoff/no stored capacity?
→ SynchronousQueue

Need transfer/handoff semantics?
→ LinkedTransferQueue
```

---

# 54. Interview Trap — `PriorityBlockingQueue` Is Not Bounded ⭐⭐⭐⭐⭐

Do not assume the name `BlockingQueue` means every implementation has a fixed capacity.

`PriorityBlockingQueue` is effectively unbounded under normal use and grows as needed until resource exhaustion.

Its `put()` does not block merely because of a normal fixed capacity limit.

---

# 55. Interview Trap — `take()` vs `poll()` ⭐⭐⭐⭐⭐

```text
take()
→ waits for an element
→ interruptible

poll()
→ returns immediately
→ null if empty
```

Timed `poll()` gives a middle ground:

```java
queue.poll(2, TimeUnit.SECONDS);
```

---

# 56. Interview Trap — `put()` vs `offer()` ⭐⭐⭐⭐⭐

```text
put()
→ waits for capacity

offer()
→ immediate result

offer(timeout)
→ bounded waiting
```

This distinction is frequently asked in Java interviews.

---

# 57. Interview Trap — Queue Thread Safety ⭐⭐⭐⭐⭐

Correct:

> The concurrent queue operations are thread-safe.

Incorrect:

> Any sequence of multiple queue operations automatically becomes atomic.

For example:

```java
if (!queue.isEmpty()) {
    queue.take();
}
```

is unnecessary and potentially racy because `take()` already expresses the required operation.

---

# 58. Practice — Avoid `isEmpty()` + `take()` ⭐⭐⭐⭐⭐

### Weak pattern

```java
if (!queue.isEmpty()) {
    queue.take();
}
```

### Better

```java
queue.take();
```

or if you specifically need non-blocking behavior:

```java
Integer value = queue.poll();
```

---

# 59. Performance Mental Model ⭐⭐⭐⭐⭐

Do not memorize one universal Big-O for all implementations.

Instead remember:

```text
ArrayBlockingQueue
→ array-backed bounded FIFO

LinkedBlockingQueue
→ linked-node FIFO

PriorityBlockingQueue
→ heap/priority based

DelayQueue
→ priority/delay based

SynchronousQueue
→ direct handoff
```

The implementation determines the performance characteristics.

---

# 60. 2-Minute Interview Answer 🏆

> **"`BlockingQueue` is a thread-safe queue abstraction from `java.util.concurrent` mainly used for producer-consumer coordination. Its important feature is that operations can block: `put()` waits when the queue is full and `take()` waits when it is empty. It also provides non-blocking operations such as `offer()` and `poll()`, plus timed variants. Common implementations are `ArrayBlockingQueue` for bounded FIFO, `LinkedBlockingQueue` for linked FIFO, `PriorityBlockingQueue` for priority ordering, `DelayQueue` for delayed availability, and `SynchronousQueue` for direct handoff. A bounded `BlockingQueue` is especially useful for applying backpressure between producers and consumers. Compared with manual `wait/notify`, it encapsulates the queue synchronization and condition waiting, reducing application-level concurrency complexity. I would choose the implementation based on ordering, capacity, and workload requirements."**

---

# 61. Quick Revision ⭐⭐⭐⭐⭐

```text
BlockingQueue
     ↓
Thread-safe producer-consumer queue
     ↓
put()  → block if full
 take() → block if empty
     ↓
offer() → false if cannot insert immediately
poll()  → null if empty
     ↓
Timed variants → bounded waiting
     ↓
ArrayBlockingQueue → bounded FIFO
LinkedBlockingQueue → linked FIFO
PriorityBlockingQueue → priority
DelayQueue → delay
SynchronousQueue → direct handoff
LinkedTransferQueue → transfer/handoff
     ↓
Bounded queue → backpressure
```

### Golden Rules

```text
add     → exception
offer   → false
put     → block

remove  → exception
poll    → null
take    → block

BlockingQueue does not mean every operation blocks.

Bounded queues can provide backpressure.

BlockingQueue is a higher-level alternative to manual wait/notify for common producer-consumer patterns.

PriorityBlockingQueue is not a normal bounded FIFO queue.

Never use isEmpty()/remainingCapacity() as a guarantee for a later concurrent operation.
```

---

# 62. 💻 Practice Checklist

- [ ] Create an `ArrayBlockingQueue`
- [ ] Practice `put()` and `take()`
- [ ] Demonstrate `put()` blocking when full
- [ ] Demonstrate `take()` blocking when empty
- [ ] Practice `offer()`
- [ ] Practice timed `offer()`
- [ ] Practice `poll()`
- [ ] Practice timed `poll()`
- [ ] Compare `add()` / `offer()` / `put()`
- [ ] Compare `remove()` / `poll()` / `take()`
- [ ] Build a producer-consumer system
- [ ] Build multiple producers/consumers
- [ ] Demonstrate bounded-queue backpressure
- [ ] Practice interrupting blocked `put()` / `take()`
- [ ] Practice `drainTo()` batching
- [ ] Practice `ArrayBlockingQueue`
- [ ] Practice `LinkedBlockingQueue`
- [ ] Practice `PriorityBlockingQueue`
- [ ] Practice `DelayQueue`
- [ ] Practice `SynchronousQueue`
- [ ] Practice `LinkedTransferQueue`
- [ ] Explain why `BlockingQueue` rejects `null`
- [ ] Explain bounded vs effectively unbounded queues
- [ ] Explain backpressure
- [ ] Explain `BlockingQueue` vs `wait/notify`
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.25 — `CopyOnWriteArrayList`](../25-CopyOnWriteArrayList/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.27 — `ArrayBlockingQueue` vs `LinkedBlockingQueue`**