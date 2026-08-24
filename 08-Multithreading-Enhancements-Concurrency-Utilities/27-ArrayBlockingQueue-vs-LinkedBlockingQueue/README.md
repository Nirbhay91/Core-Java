# 8.27 — `ArrayBlockingQueue` vs `LinkedBlockingQueue`

> **Goal:** Understand the design, capacity, memory model, locking behavior, ordering, performance trade-offs, and production use cases of the two most commonly compared `BlockingQueue` implementations.

---

## 1. Big Picture ⭐⭐⭐⭐⭐

Both are thread-safe FIFO `BlockingQueue` implementations, but their internal designs are different.

```text
ArrayBlockingQueue
    ↓
fixed-size array
    ↓
bounded capacity
    ↓
compact storage
    ↓
single lock + conditions

LinkedBlockingQueue
    ↓
linked nodes
    ↓
bounded or effectively unbounded
    ↓
separate put/take locking design
    ↓
higher flexibility
```

### Interview one-liner

> `ArrayBlockingQueue` is a fixed-capacity array-backed FIFO queue, while `LinkedBlockingQueue` is a linked-node FIFO queue whose capacity can be specified explicitly and whose implementation uses separate put and take locking mechanisms.

---

# 2. Quick Comparison ⭐⭐⭐⭐⭐

| Feature | `ArrayBlockingQueue` | `LinkedBlockingQueue` |
|---|---|---|
| Backing structure | Array | Linked nodes |
| Capacity | Fixed at construction | Configurable; effectively unbounded if omitted |
| FIFO | Yes | Yes |
| Thread-safe | Yes | Yes |
| Blocking operations | Yes | Yes |
| Memory per element | Lower overhead | Higher node overhead |
| Dynamic growth | No | Yes, up to capacity |
| Lock design | Single main lock | Separate put/take locks |
| Fairness option | Yes | No fairness constructor option |
| Good for strict bounded buffer | Excellent | Excellent when explicit capacity is supplied |
| Good for variable queue size | No | Yes |
| Null elements | Not allowed | Not allowed |

---

# 3. Creating `ArrayBlockingQueue` ⭐⭐⭐⭐⭐

```java
ArrayBlockingQueue<String> queue =
        new ArrayBlockingQueue<>(100);
```

The capacity is fixed forever for that queue instance.

You cannot resize it later.

---

# 4. Creating `LinkedBlockingQueue` ⭐⭐⭐⭐⭐

Explicitly bounded:

```java
LinkedBlockingQueue<String> queue =
        new LinkedBlockingQueue<>(100);
```

Without an explicit capacity:

```java
LinkedBlockingQueue<String> queue =
        new LinkedBlockingQueue<>();
```

The latter has an effective maximum capacity of `Integer.MAX_VALUE`; it is not literally infinite and should not be treated as unlimited-memory storage.

---

# 5. Practice — Capacity Difference ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class CapacityComparison {

    public static void main(String[] args) {
        ArrayBlockingQueue<Integer> arrayQueue =
                new ArrayBlockingQueue<>(2);

        LinkedBlockingQueue<Integer> linkedQueue =
                new LinkedBlockingQueue<>(2);

        System.out.println("Array capacity = "
                + arrayQueue.remainingCapacity());
        System.out.println("Linked capacity = "
                + linkedQueue.remainingCapacity());

        arrayQueue.add(1);
        linkedQueue.add(1);

        System.out.println("Array remaining = "
                + arrayQueue.remainingCapacity());
        System.out.println("Linked remaining = "
                + linkedQueue.remainingCapacity());
    }
}
```

---

# 6. Array-Backed Internal Model ⭐⭐⭐⭐⭐

Conceptually:

```text
ArrayBlockingQueue

[ A ][ B ][ C ][ _ ][ _ ]
  ↑              ↑
 take           put
```

The array has a fixed length.

As elements are removed and added, logical positions wrap around.

This is a **circular buffer** design.

---

# 7. Linked-Node Internal Model ⭐⭐⭐⭐⭐

Conceptually:

```text
LinkedBlockingQueue

head → Node(A) → Node(B) → Node(C) → null
                                      ↑
                                     tail
```

Each queued element is associated with a node.

Therefore, compared with an array slot, linked-node storage has additional object/reference overhead.

---

# 8. Practice — Observe FIFO Behavior ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class FifoComparison {

    public static void main(String[] args) throws InterruptedException {
        ArrayBlockingQueue<String> arrayQueue =
                new ArrayBlockingQueue<>(3);

        LinkedBlockingQueue<String> linkedQueue =
                new LinkedBlockingQueue<>(3);

        for (String value : new String[]{"A", "B", "C"}) {
            arrayQueue.put(value);
            linkedQueue.put(value);
        }

        System.out.println("Array: " + arrayQueue.take());
        System.out.println("Array: " + arrayQueue.take());
        System.out.println("Array: " + arrayQueue.take());

        System.out.println("Linked: " + linkedQueue.take());
        System.out.println("Linked: " + linkedQueue.take());
        System.out.println("Linked: " + linkedQueue.take());
    }
}
```

Both preserve FIFO ordering.

---

# 9. Locking Design ⭐⭐⭐⭐⭐

This is one of the most important differences for experienced Java interviews.

### `ArrayBlockingQueue`

Its core operations are protected by a single lock with conditions for not-empty and not-full states.

Conceptually:

```text
             one lock
                ↓
      ┌─────────┴─────────┐
      ↓                   ↓
  put / offer         take / poll
      ↓                   ↓
 notFull condition    notEmpty condition
```

### `LinkedBlockingQueue`

Its implementation uses separate locks for put-side and take-side coordination, with an atomic count coordinating queue size.

Conceptually:

```text
putLock                 takeLock
   ↓                       ↓
producers               consumers
   \                       /
    └────── count ────────┘
```

This can allow producer-side and consumer-side operations to overlap more than a single-lock design, although actual throughput depends on workload and contention.

---

# 10. Why Separate Locks Matter ⭐⭐⭐⭐⭐

Suppose:

```text
100 producer threads
100 consumer threads
```

With a single main lock, producers and consumers contend for the same lock around protected operations.

With the linked implementation's separate put/take locking structure, producer-side and consumer-side work can have more concurrency.

### Important nuance

Do **not** say:

> `LinkedBlockingQueue` is always faster.

Correct answer:

> Its locking structure can improve concurrency in suitable mixed producer-consumer workloads, but performance depends on contention, queue size, CPU, allocation pressure, workload, and fairness requirements.

---

# 11. Practice — Multiple Producers and Consumers ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerComparison {

    static void run(String name, BlockingQueue<Integer> queue)
            throws InterruptedException {

        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 1000; i++) {
                    queue.put(i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, name + "-Producer");

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 1000; i++) {
                    queue.take();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, name + "-Consumer");

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }

    public static void main(String[] args) throws InterruptedException {
        run("Array", new ArrayBlockingQueue<>(100));
        run("Linked", new LinkedBlockingQueue<>(100));
    }
}
```

This is a correctness exercise, not a reliable benchmark.

---

# 12. Fairness ⭐⭐⭐⭐⭐

`ArrayBlockingQueue` provides an optional fairness policy:

```java
new ArrayBlockingQueue<>(100, true);
```

The `true` requests fair access among waiting threads.

`LinkedBlockingQueue` does not expose the same fairness constructor option.

### Interview point

Fairness can reduce throughput because scheduling becomes more constrained.

Use it because the application needs fairness—not simply because it sounds safer.

---

# 13. Practice — Fair vs Default Array Queue ⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;

public class FairnessDemo {

    public static void main(String[] args) {
        ArrayBlockingQueue<Integer> unfair =
                new ArrayBlockingQueue<>(10);

        ArrayBlockingQueue<Integer> fair =
                new ArrayBlockingQueue<>(10, true);

        System.out.println("Default queue = " + unfair);
        System.out.println("Fair queue = " + fair);
    }
}
```

The example demonstrates construction; exact scheduling order should not be inferred from a tiny run.

---

# 14. Memory Trade-off ⭐⭐⭐⭐⭐

### `ArrayBlockingQueue`

```text
fixed array
↓
no per-element queue-node object
↓
compact representation
```

### `LinkedBlockingQueue`

```text
element
↓
linked node
↓
additional object/reference overhead
```

Therefore, when capacity is known and bounded, `ArrayBlockingQueue` can be attractive for memory predictability.

---

# 15. Allocation and Garbage Collection ⭐⭐⭐⭐

`LinkedBlockingQueue` creates and removes linked nodes as elements move through the queue.

That can introduce allocation and garbage-collection overhead under high churn.

`ArrayBlockingQueue` reuses its fixed array slots.

### Interview nuance

Do not conclude that `ArrayBlockingQueue` is always faster. Benchmark the actual workload.

---

# 16. Backpressure ⭐⭐⭐⭐⭐

Both can be bounded.

### Array

```java
new ArrayBlockingQueue<>(1000);
```

### Linked

```java
new LinkedBlockingQueue<>(1000);
```

Both can therefore provide a finite buffer and producer backpressure.

The important point is **explicit capacity**, not simply the class name.

---

# 17. Practice — Bounded Backpressure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BackpressureComparison {

    static void fill(String name, BlockingQueue<Integer> queue) {
        System.out.println(name);

        System.out.println(queue.offer(1));
        System.out.println(queue.offer(2));
        System.out.println(queue.offer(3));
        System.out.println("Remaining = "
                + queue.remainingCapacity());
    }

    public static void main(String[] args) {
        fill("Array", new ArrayBlockingQueue<>(2));
        fill("Linked", new LinkedBlockingQueue<>(2));
    }
}
```

Both reject the third immediate `offer()` because both were explicitly bounded to two elements.

---

# 18. Unbounded `LinkedBlockingQueue` Trap ⭐⭐⭐⭐⭐

This:

```java
new LinkedBlockingQueue<>();
```

does **not** mean:

```text
infinite capacity
```

It uses a very large maximum capacity.

If producers continuously outrun consumers, the queue can grow dramatically and cause memory pressure.

### Production recommendation

If workload control matters, specify a capacity:

```java
new LinkedBlockingQueue<>(1000);
```

---

# 19. Practice — Explicit Capacity Recommendation ⭐⭐⭐⭐

```java
import java.util.concurrent.LinkedBlockingQueue;

public class ExplicitCapacityDemo {

    public static void main(String[] args) {
        LinkedBlockingQueue<String> queue =
                new LinkedBlockingQueue<>(1000);

        System.out.println("Capacity available = "
                + queue.remainingCapacity());
    }
}
```

---

# 20. `remainingCapacity()` Caveat ⭐⭐⭐⭐⭐

Both implementations provide:

```java
queue.remainingCapacity();
```

But this is only a snapshot.

This is unsafe as a check-then-act guarantee:

```java
if (queue.remainingCapacity() > 0) {
    queue.put(item);
}
```

Another thread can change the queue between the check and the operation.

Prefer the actual queue operation:

```java
queue.offer(item);
```

or:

```java
queue.put(item);
```

according to the desired semantics.

---

# 21. `drainTo()` ⭐⭐⭐⭐

Both support bulk removal using `drainTo()`.

```java
List<Integer> batch = new ArrayList<>();
queue.drainTo(batch, 100);
```

This is useful for batch consumers.

It drains currently available elements; it does not wait for a batch to become full.

---

# 22. Practice — Batch Processing ⭐⭐⭐⭐⭐

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class DrainComparison {

    static void process(BlockingQueue<Integer> queue) {
        for (int i = 1; i <= 5; i++) {
            queue.offer(i);
        }

        List<Integer> batch = new ArrayList<>();
        int count = queue.drainTo(batch, 3);

        System.out.println("Drained = " + count);
        System.out.println("Batch = " + batch);
        System.out.println("Remaining = " + queue);
    }

    public static void main(String[] args) {
        System.out.println("ArrayBlockingQueue");
        process(new ArrayBlockingQueue<>(10));

        System.out.println("LinkedBlockingQueue");
        process(new LinkedBlockingQueue<>(10));
    }
}
```

---

# 23. API Similarity ⭐⭐⭐⭐⭐

Because both implement `BlockingQueue`, application code can often depend on the interface:

```java
BlockingQueue<Task> queue;
```

Then choose the implementation:

```java
queue = new ArrayBlockingQueue<>(1000);
```

or:

```java
queue = new LinkedBlockingQueue<>(1000);
```

This keeps the producer/consumer code loosely coupled to the concrete implementation.

---

# 24. Practice — Program to Interface ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProgramToInterface {

    static void produce(BlockingQueue<String> queue)
            throws InterruptedException {
        queue.put("Java");
        queue.put("Spring");
    }

    static void consume(BlockingQueue<String> queue)
            throws InterruptedException {
        System.out.println(queue.take());
        System.out.println(queue.take());
    }

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> queue =
                new ArrayBlockingQueue<>(10);

        produce(queue);
        consume(queue);
    }
}
```

---

# 25. When to Prefer `ArrayBlockingQueue` ⭐⭐⭐⭐⭐

Choose it when:

- capacity is known
- strict boundedness is important
- FIFO is required
- predictable memory usage matters
- a compact array-backed structure is desirable
- optional fairness is useful
- you want a simple bounded buffer

Example:

```text
API request → bounded work queue → workers
```

where the maximum queue size is deliberately controlled.

---

# 26. When to Prefer `LinkedBlockingQueue` ⭐⭐⭐⭐⭐

Choose it when:

- linked-node semantics fit the workload
- you want a configurable capacity
- the producer/consumer workload benefits from its separate put/take locking design
- you need a `BlockingQueue` implementation with flexible capacity configuration

Still prefer an explicit bound when unbounded growth is undesirable.

---

# 27. Production Scenario — Order Processing ⭐⭐⭐⭐⭐

Suppose:

```text
1000 incoming orders/sec
Workers process 700 orders/sec
```

A bounded queue can absorb short bursts while eventually applying backpressure.

```text
Orders
  ↓
BlockingQueue<Order>
  ↓
Worker Pool
  ↓
Payment / DB / Fulfillment
```

Possible choice:

```java
BlockingQueue<Order> queue =
        new ArrayBlockingQueue<>(5000);
```

The exact capacity should come from workload measurement, latency targets, memory limits, and failure behavior—not a magic number.

---

# 28. Practice — Order Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class OrderPipeline {

    record Order(int id) {}

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Order> queue =
                new ArrayBlockingQueue<>(3);

        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    Order order = queue.take();
                    System.out.println("Processing order " + order.id());
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        for (int i = 1; i <= 10; i++) {
            queue.put(new Order(i));
            System.out.println("Queued order " + i);
        }

        Thread.sleep(1000);
        worker.interrupt();
        worker.join();
    }
}
```

---

# 29. Do Not Benchmark Like This ⭐⭐⭐⭐⭐

Avoid conclusions such as:

```text
ArrayBlockingQueue = 20 ms
LinkedBlockingQueue = 30 ms
Therefore Array is always faster.
```

A valid concurrency benchmark must consider:

- warm-up
- JVM/JIT effects
- producer count
- consumer count
- queue capacity
- contention
- task size
- CPU architecture
- allocation/GC behavior
- fairness
- measurement methodology

For serious benchmarking, use a proper harness such as JMH.

---

# 30. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**Is `LinkedBlockingQueue` always unbounded?**

No. It can be explicitly bounded:

```java
new LinkedBlockingQueue<>(100);
```

### Trap 2
**Is `ArrayBlockingQueue` dynamically resizable?**

No. Capacity is fixed at construction.

### Trap 3
**Which one uses linked nodes?**

`LinkedBlockingQueue`.

### Trap 4
**Which one supports a fairness constructor?**

`ArrayBlockingQueue`.

### Trap 5
**Which one has separate put/take locking?**

`LinkedBlockingQueue` implementation uses separate put and take locks.

### Trap 6
**Is `LinkedBlockingQueue` always faster?**

No. Workload determines performance.

### Trap 7
**Does bounded `LinkedBlockingQueue` provide backpressure?**

Yes.

### Trap 8
**Can either queue contain `null`?**

No.

---

# 31. Deep Interview Question — Why Two Implementations? ⭐⭐⭐⭐⭐

A good answer:

> They optimize different design trade-offs. `ArrayBlockingQueue` provides a fixed-size, array-backed bounded buffer with compact storage and optional fairness. `LinkedBlockingQueue` uses linked nodes and a more concurrent put/take locking design, and supports configurable capacity. The right choice depends on boundedness, memory behavior, contention, and workload characteristics.

---

# 32. Deep Interview Question — Which Is Better? ⭐⭐⭐⭐⭐

Never answer simply:

> `ArrayBlockingQueue` is better.

or:

> `LinkedBlockingQueue` is better.

Instead:

```text
Known strict bound + compact memory
→ ArrayBlockingQueue

Configurable capacity + potentially higher producer/consumer concurrency
→ LinkedBlockingQueue

Actual production workload
→ benchmark and observe
```

---

# 33. `ArrayBlockingQueue` vs `LinkedBlockingQueue` — Mental Model 🧠

```text
                BlockingQueue
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
 ArrayBlockingQueue      LinkedBlockingQueue
          │                       │
      fixed array            linked nodes
          │                       │
      fixed capacity        configurable capacity
          │                       │
     compact storage        more node overhead
          │                       │
   single main lock       separate put/take locks
          │                       │
    optional fairness       no fairness option
```

---

# 34. 2-Minute Interview Answer 🏆

> **"Both `ArrayBlockingQueue` and `LinkedBlockingQueue` are thread-safe FIFO implementations of `BlockingQueue`. `ArrayBlockingQueue` is backed by a fixed-size circular array, so its capacity is fixed at construction. It has compact storage and supports an optional fairness policy. `LinkedBlockingQueue` uses linked nodes and allows an explicit capacity; if no capacity is supplied, its maximum is effectively `Integer.MAX_VALUE`, so I would normally specify a bound when backpressure matters. A major implementation difference is that `ArrayBlockingQueue` uses a single main lock with conditions, while `LinkedBlockingQueue` uses separate put and take locks with a count, allowing more concurrency between producer-side and consumer-side operations. I would choose ArrayBlockingQueue when I need a strict bounded, memory-predictable buffer, and LinkedBlockingQueue when its linked structure and locking characteristics fit the workload. I would not claim either is universally faster without benchmarking the actual workload."**

---

# 35. Quick Revision ⭐⭐⭐⭐⭐

```text
ArrayBlockingQueue
→ array-backed
→ fixed capacity
→ FIFO
→ compact
→ optional fairness
→ single main lock

LinkedBlockingQueue
→ linked nodes
→ configurable capacity
→ FIFO
→ more per-element overhead
→ separate put/take locks
→ no fairness constructor

Both
→ thread-safe
→ BlockingQueue
→ put/take block
→ offer/poll non-blocking
→ null not allowed
→ can provide backpressure when bounded
```

### Golden Rule

> **If the capacity is a deliberate system limit, both can be bounded. The choice is about implementation trade-offs—not simply bounded vs unbounded.**

---

# 36. 💻 Practice Checklist

- [ ] Create `ArrayBlockingQueue`
- [ ] Create bounded `LinkedBlockingQueue`
- [ ] Create default `LinkedBlockingQueue` and explain its effective maximum
- [ ] Demonstrate FIFO behavior
- [ ] Demonstrate fixed capacity
- [ ] Demonstrate backpressure
- [ ] Practice `remainingCapacity()`
- [ ] Understand the check-then-act race
- [ ] Compare fairness behavior conceptually
- [ ] Practice producer-consumer workload
- [ ] Practice multiple producers/consumers
- [ ] Practice `drainTo()`
- [ ] Explain array vs linked-node storage
- [ ] Explain single-lock vs separate put/take locking
- [ ] Explain memory overhead
- [ ] Explain allocation/GC trade-offs
- [ ] Explain when each implementation should be selected
- [ ] Explain why neither is universally faster
- [ ] Answer all interview traps
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.26 — `BlockingQueue` Implementations](../26-BlockingQueue-Implementations/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.28 — `PriorityBlockingQueue` / `DelayQueue`**