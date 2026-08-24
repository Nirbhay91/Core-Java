# 8.28 — `PriorityBlockingQueue` / `DelayQueue`

> **Goal:** Understand priority-based and delay-based blocking queues, their ordering semantics, capacity behavior, internals, production use cases, and interview traps.

---

## 1. Big Picture ⭐⭐⭐⭐⭐

Both are `BlockingQueue` implementations, but neither should be mentally modeled as a normal FIFO queue.

```text
PriorityBlockingQueue
    ↓
priority-based ordering
    ↓
smallest/highest-priority element first
    ↓
not FIFO for equal/unequal priorities

DelayQueue
    ↓
priority by delay expiration
    ↓
only expired elements can be taken
    ↓
useful for scheduled expiration/retry tasks
```

### Interview one-liner

> `PriorityBlockingQueue` is an unbounded blocking priority queue where elements are ordered by natural ordering or a comparator, while `DelayQueue` is an unbounded blocking queue of `Delayed` elements where an element becomes available to `take()` only after its delay expires.

---

# 2. Quick Comparison ⭐⭐⭐⭐⭐

| Feature | `PriorityBlockingQueue` | `DelayQueue` |
|---|---|---|
| Ordering | Priority / comparator | Delay expiration |
| FIFO | No | No |
| Blocks on empty | Yes | Yes |
| Blocks because full | No, unbounded | No, unbounded |
| Capacity | Effectively unbounded | Effectively unbounded |
| Required element type | Any comparable element or comparator | `Delayed` |
| `take()` behavior | Returns highest-priority available element | Waits until an element expires |
| Null allowed | No | No |
| Typical use | Priority work scheduling | Expiration, delayed retry, TTL tasks |

---

# 3. `PriorityBlockingQueue` Fundamentals ⭐⭐⭐⭐⭐

```java
PriorityBlockingQueue<Integer> queue =
        new PriorityBlockingQueue<>();
```

By default, elements are ordered according to natural ordering.

```java
queue.put(30);
queue.put(10);
queue.put(20);

System.out.println(queue.take()); // 10
```

The smallest integer comes out first.

---

# 4. Practice — Natural Ordering ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityNaturalOrderingDemo {

    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<Integer> queue =
                new PriorityBlockingQueue<>();

        queue.put(50);
        queue.put(10);
        queue.put(30);
        queue.put(20);

        while (!queue.isEmpty()) {
            System.out.println(queue.take());
        }
    }
}
```

Expected ordering:

```text
10
20
30
50
```

---

# 5. Max-Heap Style Priority ⭐⭐⭐⭐⭐

You can provide a comparator:

```java
PriorityBlockingQueue<Integer> queue =
        new PriorityBlockingQueue<>(11, (a, b) -> Integer.compare(b, a));
```

Now larger numbers have higher priority.

```java
queue.put(10);
queue.put(50);
queue.put(20);

System.out.println(queue.take()); // 50
```

---

# 6. Practice — Custom Comparator ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityComparatorDemo {

    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<Integer> queue =
                new PriorityBlockingQueue<>(11,
                        (a, b) -> Integer.compare(b, a));

        queue.put(10);
        queue.put(50);
        queue.put(20);
        queue.put(40);

        while (!queue.isEmpty()) {
            System.out.println(queue.take());
        }
    }
}
```

Expected:

```text
50
40
20
10
```

---

# 7. Custom Objects ⭐⭐⭐⭐⭐

```java
record Task(String name, int priority)
        implements Comparable<Task> {

    @Override
    public int compareTo(Task other) {
        return Integer.compare(this.priority, other.priority);
    }
}
```

Lower numeric priority wins in this example.

---

# 8. Practice — Priority Task Scheduler ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityTaskDemo {

    record Task(String name, int priority)
            implements Comparable<Task> {

        @Override
        public int compareTo(Task other) {
            return Integer.compare(this.priority, other.priority);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<Task> queue =
                new PriorityBlockingQueue<>();

        queue.put(new Task("Normal task", 5));
        queue.put(new Task("Critical task", 1));
        queue.put(new Task("Low task", 10));
        queue.put(new Task("High task", 2));

        while (!queue.isEmpty()) {
            Task task = queue.take();
            System.out.println(task.priority() + " -> " + task.name());
        }
    }
}
```

---

# 9. Critical Trap — `PriorityBlockingQueue` Is Unbounded ⭐⭐⭐⭐⭐

This is extremely important.

Unlike `ArrayBlockingQueue`, `PriorityBlockingQueue` does not block producers because the queue reached a normal finite capacity.

```java
queue.put(item);
```

will normally not block because of capacity.

### Therefore

Do not use `PriorityBlockingQueue` as a bounded backpressure mechanism simply because it implements `BlockingQueue`.

If producers can continuously outrun consumers, memory usage can grow significantly.

---

# 10. Practice — No Finite Capacity Backpressure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityCapacityDemo {

    public static void main(String[] args) {
        PriorityBlockingQueue<Integer> queue =
                new PriorityBlockingQueue<>(2);

        System.out.println("Remaining capacity = "
                + queue.remainingCapacity());

        queue.add(1);
        queue.add(2);
        queue.add(3);
        queue.add(4);

        System.out.println("Size = " + queue.size());
        System.out.println("Remaining capacity = "
                + queue.remainingCapacity());
    }
}
```

The constructor's initial capacity is **not a hard maximum capacity**.

---

# 11. `PriorityBlockingQueue` Ordering Nuance ⭐⭐⭐⭐⭐

The queue guarantees that the head is the least element according to its ordering.

Do not assume iteration returns elements in sorted order.

```java
for (Integer value : queue) {
    System.out.println(value);
}
```

The iterator does not guarantee priority order.

To consume according to priority:

```java
queue.take();
```

---

# 12. Practice — Iterator Trap ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityIteratorTrap {

    public static void main(String[] args) {
        PriorityBlockingQueue<Integer> queue =
                new PriorityBlockingQueue<>();

        queue.add(50);
        queue.add(10);
        queue.add(40);
        queue.add(20);

        System.out.println("Iterator order:");
        for (Integer value : queue) {
            System.out.println(value);
        }

        System.out.println("\nTake order:");
        while (!queue.isEmpty()) {
            System.out.println(queue.poll());
        }
    }
}
```

### Golden rule

> **Priority order is guaranteed at the head/removal semantics, not as a sorted iterator traversal.**

---

# 13. Equal Priority ⭐⭐⭐⭐

If two elements compare as equal, do not assume FIFO ordering between them.

For example:

```java
priority(A) == priority(B)
```

The ordering is not a stable FIFO ordering for equal-priority elements.

If stable ordering matters, add a sequence number as a tie-breaker.

---

# 14. Practice — Stable Tie-Breaker ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;
import java.util.concurrent.atomic.AtomicLong;

public class StablePriorityDemo {

    record Task(String name, int priority, long sequence)
            implements Comparable<Task> {

        @Override
        public int compareTo(Task other) {
            int priorityResult = Integer.compare(priority, other.priority);

            if (priorityResult != 0) {
                return priorityResult;
            }

            return Long.compare(sequence, other.sequence);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        AtomicLong sequence = new AtomicLong();

        PriorityBlockingQueue<Task> queue =
                new PriorityBlockingQueue<>();

        queue.put(new Task("A", 1, sequence.getAndIncrement()));
        queue.put(new Task("B", 1, sequence.getAndIncrement()));
        queue.put(new Task("C", 0, sequence.getAndIncrement()));

        while (!queue.isEmpty()) {
            System.out.println(queue.take());
        }
    }
}
```

---

# 15. `DelayQueue` Fundamentals ⭐⭐⭐⭐⭐

`DelayQueue` is a specialized `BlockingQueue`.

Its elements must implement:

```java
Delayed
```

An element is available only when:

```text
getDelay(...) <= 0
```

Until then, `take()` waits.

---

# 16. Simple `Delayed` Implementation ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

public class DelayedTask implements Delayed {

    private final String name;
    private final long triggerTime;

    public DelayedTask(String name, long delay, TimeUnit unit) {
        this.name = name;
        this.triggerTime = System.nanoTime() + unit.toNanos(delay);
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(
                triggerTime - System.nanoTime(),
                TimeUnit.NANOSECONDS);
    }

    @Override
    public int compareTo(Delayed other) {
        return Long.compare(
                this.getDelay(TimeUnit.NANOSECONDS),
                other.getDelay(TimeUnit.NANOSECONDS));
    }

    public String getName() {
        return name;
    }
}
```

### Important

Use `System.nanoTime()` for elapsed-duration calculations rather than `currentTimeMillis()`, because `nanoTime()` is intended for measuring elapsed time and is not affected by wall-clock adjustments.

---

# 17. Practice — Basic `DelayQueue` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.DelayQueue;
import java.util.concurrent.TimeUnit;

public class DelayQueueDemo {

    static class Task implements java.util.concurrent.Delayed {
        private final String name;
        private final long triggerTime;

        Task(String name, long delay, TimeUnit unit) {
            this.name = name;
            this.triggerTime = System.nanoTime() + unit.toNanos(delay);
        }

        @Override
        public long getDelay(TimeUnit unit) {
            return unit.convert(
                    triggerTime - System.nanoTime(),
                    TimeUnit.NANOSECONDS);
        }

        @Override
        public int compareTo(java.util.concurrent.Delayed other) {
            return Long.compare(
                    getDelay(TimeUnit.NANOSECONDS),
                    other.getDelay(TimeUnit.NANOSECONDS));
        }

        @Override
        public String toString() {
            return name;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        DelayQueue<Task> queue = new DelayQueue<>();

        queue.put(new Task("Task-3", 3, TimeUnit.SECONDS));
        queue.put(new Task("Task-1", 1, TimeUnit.SECONDS));
        queue.put(new Task("Task-2", 2, TimeUnit.SECONDS));

        long start = System.currentTimeMillis();

        while (!queue.isEmpty()) {
            System.out.println("Taking: " + queue.take()
                    + " at "
                    + (System.currentTimeMillis() - start) + " ms");
        }
    }
}
```

Expected conceptual order:

```text
Task-1
Task-2
Task-3
```

---

# 18. `take()` vs `poll()` in `DelayQueue` ⭐⭐⭐⭐⭐

### `take()`

Waits until an expired element is available.

```java
Task task = queue.take();
```

### `poll()`

Returns immediately with an available expired element, or `null` if none is currently available.

```java
Task task = queue.poll();
```

### Timed `poll()`

```java
Task task = queue.poll(2, TimeUnit.SECONDS);
```

---

# 19. Practice — Delay Semantics ⭐⭐⭐⭐⭐

```java
DelayQueue<Task> queue = new DelayQueue<>();

queue.put(new Task("Retry", 5, TimeUnit.SECONDS));

System.out.println(queue.poll());

Task result = queue.poll(6, TimeUnit.SECONDS);
System.out.println(result);
```

The first `poll()` can return `null` because the delay has not expired.

The timed poll can wait for an element to become available.

---

# 20. `DelayQueue` Is Not a General Scheduler ⭐⭐⭐⭐⭐

`DelayQueue` can model delayed availability, but it is not a replacement for:

- `ScheduledExecutorService`
- Quartz-like scheduling systems
- distributed schedulers
- durable job queues

It stores tasks in memory and provides delayed queue semantics.

If the JVM crashes, queued in-memory tasks are lost unless the application persists the state elsewhere.

---

# 21. Production Scenario — Retry Queue ⭐⭐⭐⭐⭐

Imagine an external service temporarily fails.

```text
API call fails
     ↓
create retry task
     ↓
DelayQueue
     ↓
retry becomes available
     ↓
worker calls API again
```

Example delays:

```text
attempt 1 → 1 second
attempt 2 → 5 seconds
attempt 3 → 30 seconds
```

In a real distributed system, persistence, duplicate handling, retry limits, and observability must also be considered.

---

# 22. Practice — Delayed Retry Worker ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.DelayQueue;
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

public class RetryWorkerDemo {

    static class RetryTask implements Delayed {
        private final String requestId;
        private final long retryAt;

        RetryTask(String requestId, long delay, TimeUnit unit) {
            this.requestId = requestId;
            this.retryAt = System.nanoTime() + unit.toNanos(delay);
        }

        @Override
        public long getDelay(TimeUnit unit) {
            return unit.convert(
                    retryAt - System.nanoTime(),
                    TimeUnit.NANOSECONDS);
        }

        @Override
        public int compareTo(Delayed other) {
            return Long.compare(
                    getDelay(TimeUnit.NANOSECONDS),
                    other.getDelay(TimeUnit.NANOSECONDS));
        }

        @Override
        public String toString() {
            return requestId;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        DelayQueue<RetryTask> queue = new DelayQueue<>();

        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    RetryTask task = queue.take();
                    System.out.println("Retrying: " + task);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        queue.put(new RetryTask("REQ-101", 2, TimeUnit.SECONDS));
        queue.put(new RetryTask("REQ-102", 1, TimeUnit.SECONDS));
        queue.put(new RetryTask("REQ-103", 4, TimeUnit.SECONDS));

        Thread.sleep(5000);
        worker.interrupt();
        worker.join();
    }
}
```

---

# 23. `DelayQueue` Ordering Internals ⭐⭐⭐⭐⭐

Conceptually:

```text
                 DelayQueue
                     ↓
                priority heap
                     ↓
             earliest deadline
                     ↓
                   head
                     ↓
         ┌───────────┴───────────┐
         ↓                       ↓
   delay <= 0              delay > 0
      ↓                         ↓
   available                not available
```

The queue uses the ordering of `Delayed` elements to determine which delayed element is at the head.

---

# 24. Why `DelayQueue.take()` Can Wait ⭐⭐⭐⭐⭐

Suppose:

```text
Task A → expires in 10 sec
Task B → expires in 20 sec
```

The head is Task A.

A consumer calling:

```java
queue.take();
```

must wait until Task A's delay expires.

It should not skip Task A merely to return Task B later, because Task B has an even later expiration.

---

# 25. `DelayQueue` and Leader/Follower Concept ⭐⭐⭐⭐

The JDK implementation uses a leader/follower optimization around the condition waiting mechanism so that one waiting consumer can time itself to the head delay while other consumers wait without all waking unnecessarily.

For interviews, remember the behavioral contract first:

> Only expired elements can be returned, and consumers block until an element becomes available.

You generally do not need to reproduce implementation source code unless specifically asked.

---

# 26. Common `Delayed` Implementation Mistake ⭐⭐⭐⭐⭐

Bad approach:

```java
return (int) (triggerTime - System.currentTimeMillis());
```

Problems include:

- wrong time unit
- integer overflow/truncation
- wall-clock adjustments
- incorrect `Delayed` contract

Prefer:

```java
@Override
public long getDelay(TimeUnit unit) {
    return unit.convert(
            triggerTime - System.nanoTime(),
            TimeUnit.NANOSECONDS);
}
```

---

# 27. Another Important Trap — `compareTo()` ⭐⭐⭐⭐⭐

`compareTo()` must be consistent with the intended expiration ordering.

A safe pattern is:

```java
@Override
public int compareTo(Delayed other) {
    return Long.compare(
            getDelay(TimeUnit.NANOSECONDS),
            other.getDelay(TimeUnit.NANOSECONDS));
}
```

Do not cast a long difference directly to `int`.

Avoid:

```java
return (int) (this.time - other.time);
```

because large differences can overflow/truncate.

---

# 28. `PriorityBlockingQueue` vs `DelayQueue` ⭐⭐⭐⭐⭐

Think:

```text
PriorityBlockingQueue
→ "Which task has the highest priority?"

DelayQueue
→ "Which delayed task is due now?"
```

Example:

```text
PriorityBlockingQueue
1. Critical payment
2. Normal payment
3. Reporting task

DelayQueue
1. Retry at 10:00:01
2. Retry at 10:00:05
3. Retry at 10:00:30
```

---

# 29. Backpressure Comparison ⭐⭐⭐⭐⭐

Neither queue provides a normal finite-capacity backpressure mechanism.

```text
ArrayBlockingQueue(100)
→ finite capacity
→ producer can block

LinkedBlockingQueue(100)
→ finite capacity
→ producer can block

PriorityBlockingQueue
→ effectively unbounded
→ no finite-capacity producer blocking

DelayQueue
→ effectively unbounded
→ no finite-capacity producer blocking
```

If you need priority ordering **and** a hard capacity limit, you may need a different design or an explicitly bounded admission layer around the priority queue.

---

# 30. Practice — Priority Work Consumer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.PriorityBlockingQueue;

public class PriorityWorkerDemo {

    record Job(String name, int priority)
            implements Comparable<Job> {

        @Override
        public int compareTo(Job other) {
            return Integer.compare(priority, other.priority);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<Job> queue =
                new PriorityBlockingQueue<>();

        Thread worker = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    Job job = queue.take();
                    System.out.println("Processing " + job);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        worker.start();

        queue.put(new Job("Normal", 5));
        queue.put(new Job("Critical", 1));
        queue.put(new Job("High", 2));

        Thread.sleep(500);
        worker.interrupt();
        worker.join();
    }
}
```

---

# 31. `PriorityBlockingQueue` vs `PriorityQueue` ⭐⭐⭐⭐⭐

`PriorityQueue`:

```java
PriorityQueue<Task> queue = new PriorityQueue<>();
```

is not designed for concurrent access.

`PriorityBlockingQueue`:

```java
PriorityBlockingQueue<Task> queue =
        new PriorityBlockingQueue<>();
```

provides thread-safe blocking queue operations.

### Important

Thread-safe does not mean every compound sequence of operations is automatically atomic.

If business logic requires multiple queue operations to behave as one transaction, additional coordination may be needed.

---

# 32. Production Selection Guide 🧠

```text
Need normal FIFO + bounded capacity
→ ArrayBlockingQueue / bounded LinkedBlockingQueue

Need priority ordering + blocking consumer
→ PriorityBlockingQueue

Need delayed availability / expiration
→ DelayQueue

Need periodic task scheduling
→ ScheduledExecutorService

Need durable distributed delayed jobs
→ external/persistent queue or scheduler
```

---

# 33. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
**Does `PriorityBlockingQueue` block when full?**

No normal finite full state exists; it is effectively unbounded.

### Trap 2
**Is `PriorityBlockingQueue` FIFO?**

No. It is priority ordered.

### Trap 3
**Does iteration return sorted order?**

No.

### Trap 4
**Does equal priority imply FIFO?**

No. Add a sequence number if stable tie-breaking is required.

### Trap 5
**What must `DelayQueue` elements implement?**

`Delayed`.

### Trap 6
**Can `DelayQueue.take()` return an unexpired task?**

No.

### Trap 7
**Is `DelayQueue` a persistent scheduler?**

No. It is an in-memory queue.

### Trap 8
**Can either queue contain `null`?**

No.

---

# 34. Senior Interview Scenario ⭐⭐⭐⭐⭐

### Question

> We need to process urgent jobs before normal jobs, but producers may generate millions of jobs. Should we use `PriorityBlockingQueue`?

### Strong answer

> `PriorityBlockingQueue` gives priority-based blocking consumption, but it is effectively unbounded, so it does not provide a hard queue-capacity backpressure mechanism. If producers can outrun consumers, memory can grow significantly. I would define the admission/backpressure strategy separately—for example, limit submissions, use a bounded front queue, reject or shed lower-priority work, or persist work externally depending on reliability requirements. I would not treat the priority queue's constructor size as a hard capacity.

---

# 35. Senior Interview Scenario — Delayed Retry ⭐⭐⭐⭐⭐

### Question

> We need retries after 1s, 5s, and 30s. Would you use `DelayQueue`?

### Strong answer

> `DelayQueue` is a good fit for an in-memory delayed-work pattern because tasks implementing `Delayed` become available only after their delay expires. However, it is not durable and not a distributed scheduler. For production retries where restart safety, horizontal scaling, persistence, or delivery guarantees matter, I would consider a durable messaging or scheduling system instead.

---

# 36. 2-Minute Interview Answer 🏆

> **"`PriorityBlockingQueue` and `DelayQueue` are both `BlockingQueue` implementations but solve different problems. `PriorityBlockingQueue` orders elements according to natural ordering or a comparator, so the highest-priority or least element is available at the head. It is effectively unbounded, so its constructor's initial capacity is not a hard maximum and it should not be treated as a backpressure mechanism. Its iterator also does not guarantee sorted order, and equal-priority elements are not guaranteed to be FIFO. `DelayQueue` is specialized for delayed availability. Its elements must implement `Delayed`, and `take()` returns an element only after its delay has expired. It is useful for in-memory expiration and delayed-retry patterns, but it is not a durable scheduler. If I need priority work, I use `PriorityBlockingQueue`; if I need delayed availability, I use `DelayQueue`; and if I need persistence, distributed scheduling, or strict bounded backpressure, I choose a different architecture."**

---

# 37. Quick Revision ⭐⭐⭐⭐⭐

```text
PriorityBlockingQueue
→ priority ordered
→ not FIFO
→ effectively unbounded
→ comparator / Comparable
→ take() blocks when empty
→ initial capacity ≠ maximum capacity
→ iterator not sorted
→ equal priority ≠ FIFO

DelayQueue
→ Delayed elements
→ ordered by delay
→ only expired elements available
→ take() waits
→ effectively unbounded
→ in-memory
→ useful for expiration / retry

Remember:
Priority = "who goes first?"
Delay = "who is due now?"
```

---

# 38. 💻 Practice Checklist

- [ ] Create `PriorityBlockingQueue`
- [ ] Use natural ordering
- [ ] Use a custom comparator
- [ ] Create a priority task object
- [ ] Demonstrate priority vs FIFO
- [ ] Demonstrate the initial-capacity trap
- [ ] Demonstrate iterator ordering trap
- [ ] Implement stable tie-breaking with a sequence number
- [ ] Implement `Delayed`
- [ ] Create a `DelayQueue`
- [ ] Demonstrate delayed `take()`
- [ ] Compare `take()` vs `poll()`
- [ ] Build a delayed retry worker
- [ ] Explain `System.nanoTime()` usage
- [ ] Explain `compareTo()` safely
- [ ] Explain why neither queue is a normal bounded backpressure queue
- [ ] Compare with `ScheduledExecutorService`
- [ ] Answer senior interview scenarios
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.27 — `ArrayBlockingQueue` vs `LinkedBlockingQueue`](../27-ArrayBlockingQueue-vs-LinkedBlockingQueue/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.29 — `ConcurrentLinkedQueue`**