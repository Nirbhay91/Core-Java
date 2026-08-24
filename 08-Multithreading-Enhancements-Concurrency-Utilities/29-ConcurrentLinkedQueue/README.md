# 8.29 — `ConcurrentLinkedQueue`

> **Goal:** Understand the lock-free, non-blocking FIFO queue provided by `ConcurrentLinkedQueue`, its CAS-based design, weakly consistent iteration, size caveats, memory behavior, and when to choose it over `BlockingQueue` implementations.

---

## 1. Big Picture ⭐⭐⭐⭐⭐

`ConcurrentLinkedQueue` is a thread-safe, **non-blocking FIFO queue** designed for concurrent access.

```text
ConcurrentLinkedQueue
        ↓
 linked nodes
        ↓
 CAS-based / lock-free algorithm
        ↓
 FIFO queue
        ↓
 operations return immediately
        ↓
 no waiting because queue is empty/full
```

### Interview one-liner

> `ConcurrentLinkedQueue` is an unbounded, thread-safe, non-blocking FIFO queue that uses a lock-free linked-node algorithm based on CAS, making it useful when producers and consumers should not block waiting for queue capacity or elements.

---

# 2. Quick Comparison ⭐⭐⭐⭐⭐

| Feature | `ConcurrentLinkedQueue` | `ArrayBlockingQueue` | `LinkedBlockingQueue` |
|---|---|---|---|
| FIFO | Yes | Yes | Yes |
| Thread-safe | Yes | Yes | Yes |
| Blocking | No | Yes | Yes |
| Lock-based waiting | No | Yes | Yes |
| Capacity | Unbounded | Fixed | Configurable |
| `put()` | ❌ | ✅ | ✅ |
| `take()` | ❌ | ✅ | ✅ |
| `offer()` | ✅ | ✅ | ✅ |
| `poll()` | ✅ | ✅ | ✅ |
| `peek()` | ✅ | ✅ | ✅ |
| `null` | Not allowed | Not allowed | Not allowed |
| Main use | Non-blocking concurrent FIFO | Bounded producer-consumer | Blocking producer-consumer |

---

# 3. Basic Creation ⭐⭐⭐⭐⭐

```java
ConcurrentLinkedQueue<String> queue =
        new ConcurrentLinkedQueue<>();
```

Add elements:

```java
queue.offer("A");
queue.offer("B");
queue.offer("C");
```

Read/remove:

```java
System.out.println(queue.poll());
```

Output:

```text
A
```

---

# 4. Practice — Basic Operations ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class BasicConcurrentLinkedQueueDemo {

    public static void main(String[] args) {
        ConcurrentLinkedQueue<String> queue =
                new ConcurrentLinkedQueue<>();

        queue.offer("Java");
        queue.offer("Spring");
        queue.offer("Kafka");

        System.out.println("Peek = " + queue.peek());
        System.out.println("Poll = " + queue.poll());
        System.out.println("Poll = " + queue.poll());
        System.out.println("Remaining = " + queue);
    }
}
```

---

# 5. Why Is It Non-Blocking? ⭐⭐⭐⭐⭐

Consider:

```java
queue.poll();
```

If the queue is empty:

```text
poll()
  ↓
returns null
```

It does **not** wait for another thread to insert an element.

Similarly:

```java
queue.offer(item);
```

returns without waiting for capacity because the queue has no fixed capacity limit.

---

# 6. Practice — Empty Queue Behavior ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class NonBlockingBehaviorDemo {

    public static void main(String[] args) {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        Integer value = queue.poll();

        System.out.println("Result = " + value);
        System.out.println("Program continues immediately.");
    }
}
```

Expected:

```text
Result = null
Program continues immediately.
```

---

# 7. `ConcurrentLinkedQueue` vs `BlockingQueue` ⭐⭐⭐⭐⭐

This is one of the most important interview comparisons.

### `ConcurrentLinkedQueue`

```java
Integer value = queue.poll();
```

Empty queue:

```text
→ null immediately
```

### `BlockingQueue`

```java
Integer value = queue.take();
```

Empty queue:

```text
→ consumer waits
```

### Mental model

```text
ConcurrentLinkedQueue
"Try now; never wait."

BlockingQueue
"Wait if the queue state requires it."
```

---

# 8. Practice — Poll vs Take ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class PollVsTakeDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> nonBlocking =
                new ConcurrentLinkedQueue<>();

        System.out.println("CLQ poll = " + nonBlocking.poll());

        LinkedBlockingQueue<Integer> blocking =
                new LinkedBlockingQueue<>();

        Thread consumer = new Thread(() -> {
            try {
                System.out.println("Waiting for blocking queue...");
                System.out.println("Taken = " + blocking.take());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        consumer.start();
        Thread.sleep(500);
        blocking.put(100);
        consumer.join();
    }
}
```

---

# 9. FIFO Ordering ⭐⭐⭐⭐⭐

`ConcurrentLinkedQueue` follows FIFO semantics.

```text
offer(A)
offer(B)
offer(C)

poll() → A
poll() → B
poll() → C
```

Concurrent execution can affect the exact interleaving of multiple producers, but the queue's removal semantics are FIFO according to the queue's concurrent ordering.

---

# 10. Practice — FIFO ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class FifoDemo {

    public static void main(String[] args) {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        queue.offer(1);
        queue.offer(2);
        queue.offer(3);

        while (!queue.isEmpty()) {
            System.out.println(queue.poll());
        }
    }
}
```

---

# 11. Multiple Producers ⭐⭐⭐⭐⭐

A major use case is allowing many threads to add work without explicitly synchronizing around a shared queue.

```text
Producer 1 ─┐
Producer 2 ─┼──→ ConcurrentLinkedQueue ──→ Consumer
Producer 3 ─┘
```

The queue itself provides thread-safe concurrent access.

---

# 12. Practice — Multiple Producers ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class MultipleProducerDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        Runnable producer = () -> {
            for (int i = 0; i < 1000; i++) {
                queue.offer(i);
            }
        };

        Thread t1 = new Thread(producer);
        Thread t2 = new Thread(producer);
        Thread t3 = new Thread(producer);

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println("Queue size = " + queue.size());
    }
}
```

The queue operations are thread-safe; the exact order of values from concurrent producers depends on their interleaving.

---

# 13. Multiple Consumers ⭐⭐⭐⭐⭐

Multiple consumers can safely call:

```java
queue.poll();
```

concurrently.

A successful `poll()` removes an element from the queue.

---

# 14. Practice — Multiple Consumers ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class MultipleConsumerDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        for (int i = 1; i <= 100; i++) {
            queue.offer(i);
        }

        Runnable consumer = () -> {
            Integer value;
            while ((value = queue.poll()) != null) {
                System.out.println(
                        Thread.currentThread().getName()
                                + " processed " + value);
            }
        };

        Thread t1 = new Thread(consumer, "Consumer-1");
        Thread t2 = new Thread(consumer, "Consumer-2");
        Thread t3 = new Thread(consumer, "Consumer-3");

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
}
```

---

# 15. Lock-Free / CAS Mental Model ⭐⭐⭐⭐⭐

The implementation uses atomic compare-and-set operations around linked nodes.

Conceptually:

```text
Thread A ─┐
          ├─ CAS → update queue links
Thread B ─┘

If another thread changed the state:
    CAS fails
        ↓
    retry
```

This avoids using a traditional mutual-exclusion lock for queue operations.

### Important terminology

For interview purposes:

> `ConcurrentLinkedQueue` is **non-blocking** and uses a **lock-free algorithm**.

Do not confuse:

```text
non-blocking
≠
lock-free
≠
wait-free
```

The algorithm's guarantee is lock-free progress, not that every individual thread completes in a bounded number of its own steps.

---

# 16. Practice — CAS Mental Model

You do not normally implement the queue yourself, but a simplified CAS-style idea looks like:

```java
// Conceptual only — not a ConcurrentLinkedQueue implementation.

Node current = tail;
Node next = current.next;

if (next == null) {
    // Conceptually attempt:
    // CAS current.next from null to newNode
}
```

The real JDK implementation contains significantly more detail and should not be reduced to this pseudocode when discussing implementation correctness.

---

# 17. Why No `put()` / `take()`? ⭐⭐⭐⭐⭐

`ConcurrentLinkedQueue` implements `Queue`, not `BlockingQueue`.

Therefore:

```java
queue.offer(item);  // yes
queue.poll();       // yes
queue.peek();       // yes
```

but:

```java
queue.put(item);    // no
queue.take();       // no
```

---

# 18. `offer()` vs `add()` ⭐⭐⭐⭐

For `ConcurrentLinkedQueue`, both can insert successfully under normal operation because the queue is unbounded.

```java
queue.offer("A");
queue.add("B");
```

Still, `offer()` often communicates queue semantics more clearly.

---

# 19. `poll()` vs `remove()` ⭐⭐⭐⭐⭐

```java
queue.poll();
```

returns:

```text
head element
or null if empty
```

Whereas:

```java
queue.remove();
```

throws `NoSuchElementException` if empty.

### Interview recommendation

For concurrent queue consumption where empty is an expected state, `poll()` is usually the clearer choice.

---

# 20. Practice — `poll()` vs `remove()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class PollVsRemoveDemo {

    public static void main(String[] args) {
        ConcurrentLinkedQueue<String> queue =
                new ConcurrentLinkedQueue<>();

        System.out.println("poll = " + queue.poll());

        try {
            queue.remove();
        } catch (Exception e) {
            System.out.println("remove failed: "
                    + e.getClass().getSimpleName());
        }
    }
}
```

---

# 21. `peek()` ⭐⭐⭐⭐

`peek()` reads the head without removing it.

```java
String value = queue.peek();
```

If empty:

```text
null
```

### Concurrency warning

This is not a reservation operation.

Another thread can remove the element immediately after `peek()` returns.

---

# 22. The Check-Then-Act Trap ⭐⭐⭐⭐⭐

Avoid reasoning like:

```java
if (!queue.isEmpty()) {
    return queue.poll();
}
```

Another consumer can remove the element between:

```text
isEmpty()
   ↓
context switch
   ↓
poll()
```

So `poll()` can still return `null`.

Prefer directly using the atomic queue operation:

```java
Integer value = queue.poll();
if (value != null) {
    process(value);
}
```

---

# 23. Practice — Correct Consumer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class CorrectConsumerDemo {

    static void consume(ConcurrentLinkedQueue<String> queue) {
        String item;

        while ((item = queue.poll()) != null) {
            System.out.println("Processing " + item);
        }
    }

    public static void main(String[] args) {
        ConcurrentLinkedQueue<String> queue =
                new ConcurrentLinkedQueue<>();

        queue.add("A");
        queue.add("B");
        queue.add("C");

        consume(queue);
    }
}
```

---

# 24. `size()` Is Expensive / Weak Under Concurrency ⭐⭐⭐⭐⭐

This is a very important interview point.

```java
queue.size();
```

is not a cheap constant-time capacity counter you should rely on for high-performance concurrent control logic.

The size may require traversal, and under concurrent modification the observed value can become stale immediately.

### Therefore

Avoid:

```java
if (queue.size() > 0) {
    queue.poll();
}
```

Use:

```java
Object item = queue.poll();
```

and handle the result.

---

# 25. Practice — Size Snapshot Trap ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class SizeSnapshotDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        queue.add(1);
        queue.add(2);
        queue.add(3);

        Thread consumer = new Thread(() -> {
            queue.poll();
            queue.poll();
        });

        int observed = queue.size();
        consumer.start();
        consumer.join();

        System.out.println("Observed earlier = " + observed);
        System.out.println("Current size = " + queue.size());
    }
}
```

The earlier size is only an observation at that point in time.

---

# 26. `isEmpty()` Under Concurrency ⭐⭐⭐⭐

`isEmpty()` is useful for observation, but do not use it as a synchronization guarantee.

Bad pattern:

```java
while (!queue.isEmpty()) {
    process(queue.poll());
}
```

Another thread can drain the queue between the check and the poll.

Better:

```java
Integer item;
while ((item = queue.poll()) != null) {
    process(item);
}
```

---

# 27. Weakly Consistent Iterator ⭐⭐⭐⭐⭐

Iterators of concurrent collections are designed to tolerate concurrent modification.

For `ConcurrentLinkedQueue`, the iterator is **weakly consistent**.

It:

- does not throw `ConcurrentModificationException` merely because another thread modifies the queue
- may reflect some elements added after iteration starts
- may not reflect every concurrent modification
- does not provide a frozen snapshot

---

# 28. Practice — Concurrent Iteration ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class WeakIteratorDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        for (int i = 1; i <= 5; i++) {
            queue.add(i);
        }

        Thread producer = new Thread(() -> {
            for (int i = 6; i <= 10; i++) {
                queue.add(i);
            }
        });

        producer.start();

        for (Integer value : queue) {
            System.out.println(value);
        }

        producer.join();
    }
}
```

Do not depend on one exact output ordering for the concurrent portion.

---

# 29. Memory Behavior ⭐⭐⭐⭐

`ConcurrentLinkedQueue` is linked-node based.

Conceptually:

```text
head → Node(A) → Node(B) → Node(C) → null
```

Therefore each queued element has node/reference overhead.

Because the queue is unbounded, sustained producer-over-consumer imbalance can create significant memory pressure.

---

# 30. No Built-In Backpressure ⭐⭐⭐⭐⭐

This is a major production concern.

```text
Producer rate > Consumer rate
        ↓
queue grows
        ↓
memory grows
        ↓
possible GC pressure
        ↓
possible OutOfMemoryError
```

If you need a hard capacity boundary, prefer a bounded `BlockingQueue` or introduce an explicit admission-control mechanism.

---

# 31. Practice — Producer Faster Than Consumer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class GrowthRiskDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        Thread producer = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                queue.offer(i);
            }
        });

        producer.start();
        producer.join();

        System.out.println("Queue size = " + queue.size());
        System.out.println("Consumer must catch up.");
    }
}
```

This demonstrates why an unbounded queue needs an explicit workload strategy.

---

# 32. `remove(Object)` ⭐⭐⭐⭐

`ConcurrentLinkedQueue` supports removing a matching element:

```java
queue.remove("Java");
```

But this is not the same operation as:

```java
queue.poll();
```

`poll()` removes the head.

`remove(Object)` searches for a matching element.

Avoid relying on arbitrary middle removal in a hot path unless the workload actually needs it.

---

# 33. Practice — Remove Specific Element ⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class RemoveSpecificDemo {

    public static void main(String[] args) {
        ConcurrentLinkedQueue<String> queue =
                new ConcurrentLinkedQueue<>();

        queue.add("Java");
        queue.add("Spring");
        queue.add("Kafka");

        boolean removed = queue.remove("Spring");

        System.out.println("Removed = " + removed);
        System.out.println("Queue = " + queue);
    }
}
```

---

# 34. Use Case — Event Handoff ⭐⭐⭐⭐⭐

A useful pattern:

```text
Thread / callback
      ↓
ConcurrentLinkedQueue<Event>
      ↓
consumer loop
      ↓
process events
```

This is suitable when the consumer can poll without needing to wait inside the queue itself.

If waiting is required, a `BlockingQueue` is often a better fit.

---

# 35. Practice — Event Queue ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class EventQueueDemo {

    record Event(String type, String payload) {}

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Event> queue =
                new ConcurrentLinkedQueue<>();

        Thread producer = new Thread(() -> {
            queue.offer(new Event("LOGIN", "user-101"));
            queue.offer(new Event("PAYMENT", "order-500"));
            queue.offer(new Event("LOGOUT", "user-101"));
        });

        Thread consumer = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                Event event = queue.poll();

                if (event != null) {
                    System.out.println("Processing: " + event);
                } else {
                    Thread.yield();
                }

                if (queue.isEmpty() && !producer.isAlive()) {
                    break;
                }
            }
        });

        producer.start();
        consumer.start();

        producer.join();
        consumer.join();
    }
}
```

### Production caution

A loop that repeatedly polls and yields can still waste CPU. For a consumer that should wait efficiently for work, prefer a blocking mechanism.

---

# 36. Busy Polling vs Blocking ⭐⭐⭐⭐⭐

### Busy/non-blocking approach

```java
while (...) {
    Item item = queue.poll();
    if (item != null) {
        process(item);
    }
}
```

Potential issue:

```text
empty queue
   ↓
poll
poll
poll
poll
...
CPU usage
```

### Blocking approach

```java
Item item = blockingQueue.take();
```

The thread can wait efficiently for work.

### Interview answer

> `ConcurrentLinkedQueue` is useful when non-blocking progress is desired; it is not automatically the right choice for a producer-consumer worker queue where consumers should sleep while no work exists.

---

# 37. `ConcurrentLinkedQueue` vs `ConcurrentLinkedDeque` ⭐⭐⭐⭐

`ConcurrentLinkedQueue`:

```text
FIFO queue
```

`ConcurrentLinkedDeque`:

```text
FIFO + operations at both ends
```

Use the deque when the algorithm genuinely requires adding/removing from both ends.

---

# 38. `ConcurrentLinkedQueue` vs `CopyOnWriteArrayList` ⭐⭐⭐⭐

Do not confuse their use cases.

```text
ConcurrentLinkedQueue
→ concurrent queue / producer-consumer style access

CopyOnWriteArrayList
→ read-heavy list with infrequent writes
```

`CopyOnWriteArrayList` copies the underlying array on writes, while `ConcurrentLinkedQueue` uses linked nodes and lock-free queue operations.

---

# 39. `ConcurrentLinkedQueue` vs `PriorityBlockingQueue` ⭐⭐⭐⭐⭐

```text
ConcurrentLinkedQueue
→ FIFO
→ non-blocking
→ unbounded

PriorityBlockingQueue
→ priority ordering
→ blocking take when empty
→ unbounded
```

The important difference is both **ordering** and **blocking behavior**.

---

# 40. Practice — Compare Three Queues ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.PriorityBlockingQueue;

public class QueueComparisonDemo {

    public static void main(String[] args) throws Exception {
        ConcurrentLinkedQueue<Integer> clq =
                new ConcurrentLinkedQueue<>();

        LinkedBlockingQueue<Integer> blocking =
                new LinkedBlockingQueue<>(10);

        PriorityBlockingQueue<Integer> priority =
                new PriorityBlockingQueue<>();

        clq.offer(30);
        clq.offer(10);
        clq.offer(20);

        blocking.put(30);
        blocking.put(10);
        blocking.put(20);

        priority.put(30);
        priority.put(10);
        priority.put(20);

        System.out.println("CLQ: " + clq.poll());
        System.out.println("BlockingQueue: " + blocking.take());
        System.out.println("PriorityBlockingQueue: " + priority.take());
    }
}
```

Expected first removals:

```text
CLQ → 30
BlockingQueue → 30
PriorityBlockingQueue → 10
```

---

# 41. Thread Safety vs Compound Operations ⭐⭐⭐⭐⭐

The queue's individual operations are thread-safe.

But this sequence:

```java
if (!queue.isEmpty()) {
    Item item = queue.poll();
    // business action
}
```

is not one atomic transaction.

Similarly:

```java
if (queue.contains(item)) {
    queue.remove(item);
}
```

can race with another thread.

### Rule

> Thread-safe collection operations do not automatically make a multi-operation business workflow atomic.

---

# 42. Practice — Compound Operation Trap ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class CompoundOperationTrap {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<String> queue =
                new ConcurrentLinkedQueue<>();

        queue.add("TASK");

        Thread consumer1 = new Thread(() -> {
            if (!queue.isEmpty()) {
                System.out.println("C1 = " + queue.poll());
            }
        });

        Thread consumer2 = new Thread(() -> {
            if (!queue.isEmpty()) {
                System.out.println("C2 = " + queue.poll());
            }
        });

        consumer1.start();
        consumer2.start();

        consumer1.join();
        consumer2.join();
    }
}
```

Possible output includes one consumer receiving `TASK` and the other receiving `null` despite its earlier `isEmpty()` check.

---

# 43. Performance Thinking ⭐⭐⭐⭐⭐

Do not say:

> `ConcurrentLinkedQueue` is faster than `LinkedBlockingQueue`.

A better answer:

> `ConcurrentLinkedQueue` avoids blocking and uses a lock-free algorithm, which can be beneficial under suitable contention patterns. `LinkedBlockingQueue` provides blocking semantics and bounded capacity when configured, which may be more appropriate for worker pipelines. Performance depends on the workload and the cost of waiting, contention, polling, allocation, and downstream processing.

---

# 44. When to Choose `ConcurrentLinkedQueue` ⭐⭐⭐⭐⭐

Choose it when:

- FIFO ordering is required
- thread-safe concurrent access is required
- operations should not block waiting for elements
- an unbounded queue is acceptable
- polling/try-now semantics fit the design
- you want a lock-free queue implementation

Examples:

```text
non-blocking event handoff
concurrent work accumulation
internal queues where empty means "nothing to do"
low-level concurrent algorithms
```

---

# 45. When NOT to Choose It ⭐⭐⭐⭐⭐

Avoid it when you need:

- bounded capacity
- producer backpressure
- `take()` semantics
- automatic waiting for new work
- timed blocking operations

Prefer:

```java
BlockingQueue<T>
```

for those requirements.

---

# 46. Production Scenario ⭐⭐⭐⭐⭐

### Requirement

> Multiple callback threads generate events. A processing thread periodically drains available events. If no event exists, it can perform other work instead of blocking on the queue.

Good candidate:

```java
ConcurrentLinkedQueue<Event> queue =
        new ConcurrentLinkedQueue<>();
```

But if the processor should wait efficiently for events:

```java
BlockingQueue<Event> queue =
        new LinkedBlockingQueue<>(1000);
```

may be a better design.

---

# 47. Senior Interview Scenario 🏆

### Question

> Why would you use `ConcurrentLinkedQueue` instead of `LinkedBlockingQueue`?

### Strong answer

> I would choose `ConcurrentLinkedQueue` when I specifically need non-blocking FIFO operations and an unbounded queue is acceptable. An empty `poll()` returns immediately instead of making the consumer wait. It uses a lock-free linked-node algorithm based on CAS, so it avoids traditional lock contention. I would choose `LinkedBlockingQueue` when the consumer should wait for work or when I need a bounded queue to provide backpressure. The choice is primarily about semantics and workload, not simply which implementation is faster.

---

# 48. Senior Interview Scenario — Memory Risk 🏆

### Question

> Producers are generating 50,000 tasks/sec and consumers process 30,000/sec. Is `ConcurrentLinkedQueue` safe?

### Strong answer

> The queue is thread-safe, but thread safety does not solve capacity management. Because `ConcurrentLinkedQueue` is unbounded, the backlog can continuously grow by roughly 20,000 tasks/sec during that sustained imbalance. That creates memory and GC pressure and can eventually exhaust the JVM. I would introduce bounded backpressure, rate limiting, load shedding, persistence, or additional consumers depending on the business requirement.

---

# 49. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**Is `ConcurrentLinkedQueue` blocking?**

No.

### Trap 2
**Does `poll()` wait for an element?**

No. It returns `null` if empty.

### Trap 3
**Does it support `take()`?**

No.

### Trap 4
**Is it bounded?**

No. It is unbounded.

### Trap 5
**Does it use a traditional synchronized lock around every operation?**

No. It uses a lock-free algorithm based on atomic/CAS operations.

### Trap 6
**Is `size()` a good synchronization mechanism?**

No.

### Trap 7
**Does `isEmpty()` guarantee the next `poll()` succeeds?**

No.

### Trap 8
**Does thread-safe mean compound operations are atomic?**

No.

### Trap 9
**Does it provide backpressure?**

No built-in finite-capacity backpressure.

### Trap 10
**Is it always faster than blocking queues?**

No.

---

# 50. 2-Minute Interview Answer 🏆

> **"`ConcurrentLinkedQueue` is a thread-safe, unbounded, non-blocking FIFO queue. It uses a lock-free linked-node algorithm based on CAS, so producers and consumers can concurrently perform queue operations without using a traditional mutual-exclusion lock. Unlike a `BlockingQueue`, it does not have `put()` or `take()` semantics. `poll()` returns immediately with the head element or `null` if the queue is empty. Because it is unbounded, it does not provide built-in finite-capacity backpressure, so a sustained producer-consumer imbalance can cause memory growth. Its iterator is weakly consistent, and methods such as `size()` should not be used as synchronization guarantees under concurrent modification. I would use it when non-blocking FIFO access is required and unbounded growth is acceptable. If I need consumers to wait for work or need bounded backpressure, I would choose an appropriate `BlockingQueue` instead."**

---

# 51. Quick Revision ⭐⭐⭐⭐⭐

```text
ConcurrentLinkedQueue
        ↓
thread-safe
        ↓
FIFO
        ↓
unbounded
        ↓
non-blocking
        ↓
lock-free / CAS-based
        ↓
offer / poll / peek
        ↓
NO put / take
        ↓
NO built-in backpressure
        ↓
weakly consistent iterator
        ↓
size() not a synchronization tool
```

### Golden Rule 🧠

> **`ConcurrentLinkedQueue` = "concurrent + FIFO + try-now semantics"; `BlockingQueue` = "concurrent + FIFO/other ordering + wait when required."**

---

# 52. 💻 Practice Checklist

- [ ] Create a `ConcurrentLinkedQueue`
- [ ] Practice `offer()`
- [ ] Practice `poll()`
- [ ] Practice `peek()`
- [ ] Compare `poll()` with `take()`
- [ ] Demonstrate FIFO
- [ ] Use multiple producers
- [ ] Use multiple consumers
- [ ] Understand CAS / lock-free mental model
- [ ] Explain non-blocking semantics
- [ ] Explain why there is no `put()` / `take()`
- [ ] Practice `poll()` vs `remove()`
- [ ] Understand `peek()` race behavior
- [ ] Understand `size()` caveat
- [ ] Understand `isEmpty()` check-then-act race
- [ ] Practice weakly consistent iteration
- [ ] Understand unbounded memory risk
- [ ] Compare with `LinkedBlockingQueue`
- [ ] Compare with `PriorityBlockingQueue`
- [ ] Explain compound-operation thread safety
- [ ] Answer production scenarios
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.28 — `PriorityBlockingQueue` / `DelayQueue`](../28-PriorityBlockingQueue-and-DelayQueue/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.30 — `CompletableFuture` Fundamentals**