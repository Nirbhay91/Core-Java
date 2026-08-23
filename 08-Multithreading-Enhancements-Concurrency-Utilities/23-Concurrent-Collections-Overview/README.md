# 8.23 — Concurrent Collections Overview

> **Goal:** Understand why concurrent collections exist, how they differ from ordinary collections, which collection to choose for a concurrent workload, and the core trade-offs between locking, weakly consistent iteration, blocking behavior, and copy-on-write strategies.

---

## 1. Why Concurrent Collections? ⭐⭐⭐⭐⭐

Ordinary collections such as:

```java
ArrayList
HashMap
HashSet
LinkedList
```

are generally **not designed for concurrent structural modification by multiple threads**.

Example of the problem:

```text
Thread A → HashMap.put()
Thread B → HashMap.put()
Thread C → HashMap.get()
```

Without appropriate coordination, the application can have race conditions or unsafe visibility/iteration behavior.

Java provides concurrent collections in:

```java
java.util.concurrent
java.util.concurrent.atomic
```

Examples:

```text
ConcurrentHashMap
CopyOnWriteArrayList
CopyOnWriteArraySet
BlockingQueue implementations
ConcurrentLinkedQueue
ConcurrentSkipListMap
ConcurrentSkipListSet
```

---

# 2. The Main Categories ⭐⭐⭐⭐⭐

Think of concurrent collections in four groups:

```text
Concurrent Collections
        │
        ├── Concurrent Maps/Sets
        │      ├── ConcurrentHashMap
        │      ├── ConcurrentSkipListMap
        │      └── ConcurrentSkipListSet
        │
        ├── Copy-On-Write
        │      ├── CopyOnWriteArrayList
        │      └── CopyOnWriteArraySet
        │
        ├── Blocking Queues
        │      ├── ArrayBlockingQueue
        │      ├── LinkedBlockingQueue
        │      ├── PriorityBlockingQueue
        │      └── DelayQueue
        │
        └── Non-blocking Queues
               └── ConcurrentLinkedQueue
```

---

# 3. Quick Selection Table ⭐⭐⭐⭐⭐

| Requirement | Preferred Collection |
|---|---|
| Concurrent key-value map | `ConcurrentHashMap` |
| Concurrent sorted map | `ConcurrentSkipListMap` |
| Concurrent sorted set | `ConcurrentSkipListSet` |
| Many reads, very few writes | `CopyOnWriteArrayList` |
| Producer-consumer with waiting | `BlockingQueue` |
| Bounded producer-consumer queue | `ArrayBlockingQueue` |
| Optionally bounded linked queue | `LinkedBlockingQueue` |
| Priority-based blocking queue | `PriorityBlockingQueue` |
| Delay-based scheduling queue | `DelayQueue` |
| High-throughput non-blocking FIFO queue | `ConcurrentLinkedQueue` |

---

# 4. `ConcurrentHashMap` ⭐⭐⭐⭐⭐

`ConcurrentHashMap` is the primary concurrent map used when multiple threads need to access and update a map safely.

```java
import java.util.concurrent.ConcurrentHashMap;

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("Java", 10);
map.put("Spring", 20);

System.out.println(map.get("Java"));
```

It supports concurrent access without requiring the caller to synchronize every individual operation.

---

# 5. Practice — Concurrent Map Updates ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentHashMapBasic {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> map =
                new ConcurrentHashMap<>();

        Runnable task = () -> {
            for (int i = 0; i < 1_000; i++) {
                map.merge("Java", 1, Integer::sum);
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        Thread t3 = new Thread(task);

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println(map.get("Java")); // 3000
    }
}
```

### Interview Point ⭐⭐⭐⭐⭐

`merge()` performs the update as a map-level atomic operation for the relevant key. This is much safer than:

```java
map.put("Java", map.get("Java") + 1);
```

when multiple threads update the same key.

---

# 6. `ConcurrentHashMap` Does Not Allow `null` Keys/Values ⭐⭐⭐⭐⭐

This is an important interview question.

```java
ConcurrentHashMap<String, String> map =
        new ConcurrentHashMap<>();

map.put(null, "value"); // NullPointerException
```

and:

```java
map.put("key", null); // NullPointerException
```

The absence of `null` avoids ambiguity between:

```text
key does not exist
        vs
key exists with null value
```

---

# 7. `CopyOnWriteArrayList` ⭐⭐⭐⭐⭐

`CopyOnWriteArrayList` creates a new underlying array when the collection is modified.

```java
import java.util.concurrent.CopyOnWriteArrayList;

CopyOnWriteArrayList<String> list =
        new CopyOnWriteArrayList<>();

list.add("Java");
list.add("Spring");
```

Mental model:

```text
READ READ READ READ
   ↓
existing array

WRITE
   ↓
copy array
   ↓
modify copy
   ↓
swap/reference update
```

This makes it particularly useful for **read-heavy, write-light** workloads.

---

# 8. Practice — Safe Iteration with Copy-On-Write ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class CopyOnWriteBasic {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("Java");
        list.add("Spring");
        list.add("Kafka");

        for (String item : list) {
            System.out.println(item);
            list.add("New Item");
        }

        System.out.println(list);
    }
}
```

The iterator works over the snapshot it obtained, so concurrent structural modification does not produce the usual `ConcurrentModificationException` behavior of a standard `ArrayList` iterator.

⚠️ Do not interpret this as "all operations are universally atomic." The collection's thread-safe operations and iterator semantics must still be understood individually.

---

# 9. `CopyOnWriteArrayList` Trade-Off ⭐⭐⭐⭐⭐

Advantages:

```text
Excellent concurrent reads
Snapshot-style iteration
No reader lock for normal reads
Simple iteration semantics
```

Disadvantages:

```text
Every write copies the array
More memory allocation
Expensive for frequent writes
Not suitable for large write-heavy lists
```

### Best Mental Model

> **Many readers + very few writers → consider Copy-On-Write.**

Typical examples:

```text
configuration listeners
observer lists
registered callbacks
mostly-static configuration
```

---

# 10. `BlockingQueue` ⭐⭐⭐⭐⭐

A `BlockingQueue` is useful for producer-consumer designs.

```text
Producer
   ↓
put()
   ↓
BlockingQueue
   ↓
take()
   ↓
Consumer
```

Important operations include:

```java
put()
take()
offer()
poll()
```

Blocking behavior:

```text
put()  → waits if bounded queue is full

take() → waits if queue is empty
```

---

# 11. Practice — Producer Consumer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerBasic {

    public static void main(String[] args) {
        BlockingQueue<Integer> queue =
                new ArrayBlockingQueue<>(2);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    queue.put(i);
                    System.out.println("Produced: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    int value = queue.take();
                    System.out.println("Consumed: " + value);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

---

# 12. `ArrayBlockingQueue` ⭐⭐⭐⭐⭐

Characteristics:

```text
bounded
array-backed
FIFO
blocking
```

Creation:

```java
BlockingQueue<String> queue =
        new ArrayBlockingQueue<>(100);
```

The capacity is fixed.

Use it when you need explicit boundedness and predictable memory usage.

---

# 13. `LinkedBlockingQueue` ⭐⭐⭐⭐⭐

Characteristics:

```text
linked-node based
FIFO
blocking
optionally bounded
```

Example:

```java
BlockingQueue<String> queue =
        new java.util.concurrent.LinkedBlockingQueue<>(100);
```

It is commonly used in producer-consumer pipelines and executor implementations.

---

# 14. `PriorityBlockingQueue` ⭐⭐⭐⭐⭐

Unlike normal FIFO queues, it orders elements according to priority/comparator.

```java
import java.util.concurrent.PriorityBlockingQueue;

PriorityBlockingQueue<Integer> queue =
        new PriorityBlockingQueue<>();

queue.put(30);
queue.put(10);
queue.put(20);

System.out.println(queue.take()); // 10
```

Important:

> `PriorityBlockingQueue` is unbounded, so it should not be treated as a bounded backpressure mechanism.

---

# 15. `DelayQueue` ⭐⭐⭐⭐

A `DelayQueue` contains elements that become available only after their delay expires.

It is useful for concepts such as:

```text
scheduled expiration
retry-after delay
temporary cache entries
delayed tasks
```

It is unbounded and elements must implement `Delayed`.

---

# 16. `ConcurrentLinkedQueue` ⭐⭐⭐⭐⭐

`ConcurrentLinkedQueue` is a non-blocking concurrent FIFO queue.

```java
import java.util.concurrent.ConcurrentLinkedQueue;

ConcurrentLinkedQueue<String> queue =
        new ConcurrentLinkedQueue<>();

queue.offer("A");
queue.offer("B");

System.out.println(queue.poll()); // A
```

Key difference:

```text
BlockingQueue
→ may wait

ConcurrentLinkedQueue
→ does not wait for an element to become available
```

---

# 17. Practice — ConcurrentLinkedQueue ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class ConcurrentLinkedQueueBasic {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue =
                new ConcurrentLinkedQueue<>();

        Runnable producer = () -> {
            for (int i = 0; i < 1_000; i++) {
                queue.offer(i);
            }
        };

        Thread t1 = new Thread(producer);
        Thread t2 = new Thread(producer);

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println("Size = " + queue.size());
    }
}
```

⚠️ In highly concurrent situations, avoid designing algorithms that repeatedly depend on `size()` as an exact coordination decision. Prefer queue operations themselves.

---

# 18. `ConcurrentSkipListMap` ⭐⭐⭐⭐

When you need:

```text
concurrent map
+
sorted keys
```

consider:

```java
ConcurrentSkipListMap<Integer, String> map =
        new ConcurrentSkipListMap<>();
```

Example:

```java
map.put(30, "C");
map.put(10, "A");
map.put(20, "B");

System.out.println(map.firstKey()); // 10
```

It provides a concurrent sorted-map structure.

---

# 19. `ConcurrentSkipListSet` ⭐⭐⭐⭐

For a concurrent sorted set:

```java
ConcurrentSkipListSet<Integer> set =
        new ConcurrentSkipListSet<>();

set.add(30);
set.add(10);
set.add(20);

System.out.println(set.first()); // 10
```

Mental shortcut:

```text
ConcurrentHashMap
→ concurrent + generally unordered map

ConcurrentSkipListMap
→ concurrent + sorted map

ConcurrentSkipListSet
→ concurrent + sorted set
```

---

# 20. Fail-Fast vs Weakly Consistent vs Snapshot Iterators ⭐⭐⭐⭐⭐

This is an important interview topic.

### Ordinary collection iterator

Often fail-fast on structural modification:

```text
ArrayList / HashMap
       ↓
structural modification during iteration
       ↓
ConcurrentModificationException may occur
```

### Concurrent collections

Many provide **weakly consistent** iterators:

```text
ConcurrentHashMap
ConcurrentLinkedQueue
```

The iterator can proceed while other threads modify the collection and does not generally throw `ConcurrentModificationException` merely because of those concurrent modifications.

### Copy-on-write collections

Provide snapshot-style iteration:

```text
CopyOnWriteArrayList
        ↓
iterator sees the array snapshot captured by iterator creation
```

---

# 21. Weakly Consistent Does NOT Mean Snapshot ⭐⭐⭐⭐⭐

Important distinction:

```text
Weakly consistent
→ may reflect some updates
→ may not reflect all concurrent updates
→ does not provide a frozen snapshot
```

Do not say:

> "ConcurrentHashMap iterator always gives a snapshot."

That is incorrect.

---

# 22. Practice — ConcurrentHashMap Iteration ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentMapIteration {

    public static void main(String[] args) {
        ConcurrentHashMap<Integer, String> map =
                new ConcurrentHashMap<>();

        map.put(1, "Java");
        map.put(2, "Spring");

        for (Integer key : map.keySet()) {
            System.out.println(key + " = " + map.get(key));
            map.put(3, "Kafka");
        }

        System.out.println(map);
    }
}
```

The important lesson is not the exact printed order/content, but that concurrent map iteration has different semantics from ordinary fail-fast collection iteration.

---

# 23. Compound Actions Are Still Important ⚠️⭐⭐⭐⭐⭐

Thread-safe collection methods do not automatically make every multi-step algorithm thread-safe.

Unsafe pattern:

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

Another thread can modify the map between the two operations.

Prefer atomic compound APIs when available:

```java
map.putIfAbsent(key, value);
```

or:

```java
map.computeIfAbsent(key, k -> createValue(k));
```

---

# 24. Practice — `putIfAbsent()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentHashMap;

public class PutIfAbsentExample {

    public static void main(String[] args) {
        ConcurrentHashMap<String, String> map =
                new ConcurrentHashMap<>();

        map.putIfAbsent("Java", "Backend");
        map.putIfAbsent("Java", "Language");

        System.out.println(map.get("Java")); // Backend
    }
}
```

---

# 25. Practice — `computeIfAbsent()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ConcurrentHashMap;

public class ComputeIfAbsentExample {

    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> counts =
                new ConcurrentHashMap<>();

        counts.computeIfAbsent("Java", key -> 0);
        counts.compute("Java", (key, value) -> value + 1);

        System.out.println(counts.get("Java")); // 1
    }
}
```

---

# 26. Concurrent Collection ≠ Lock-Free Collection ⭐⭐⭐⭐⭐

Do not treat these terms as synonyms.

```text
Thread-safe
→ operations are safe according to API semantics

Concurrent
→ designed for useful concurrent access

Lock-free
→ specific progress guarantee

Blocking
→ operation may wait
```

For example:

```text
ConcurrentHashMap
→ concurrent collection

ArrayBlockingQueue
→ concurrent + blocking

ConcurrentLinkedQueue
→ concurrent + non-blocking algorithm
```

---

# 27. `Collections.synchronizedList()` vs Concurrent Collections ⭐⭐⭐⭐⭐

You can wrap a collection:

```java
List<String> list =
        Collections.synchronizedList(new ArrayList<>());
```

This provides synchronized access to individual operations.

But it is not automatically equivalent to a purpose-built concurrent collection.

For iteration, external synchronization is commonly required:

```java
synchronized (list) {
    for (String value : list) {
        System.out.println(value);
    }
}
```

A purpose-built concurrent collection may provide better concurrency and different iterator semantics.

---

# 28. Practice — Synchronized List Iteration ⭐⭐⭐⭐

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class SynchronizedListIteration {

    public static void main(String[] args) {
        List<String> list = Collections.synchronizedList(
                new ArrayList<>());

        list.add("Java");
        list.add("Spring");

        synchronized (list) {
            for (String item : list) {
                System.out.println(item);
            }
        }
    }
}
```

---

# 29. Choosing the Right Collection — Interview Decision Tree ⭐⭐⭐⭐⭐

```text
Need key-value map?
   │
   ├── Yes → ConcurrentHashMap
   │            │
   │            └── Need sorted keys?
   │                    ↓
   │              ConcurrentSkipListMap
   │
   └── No
       │
       Need list?
       │
       ├── Mostly reads, few writes?
       │       ↓
       │   CopyOnWriteArrayList
       │
       └── Need producer-consumer?
               │
               ├── Need blocking?
               │      ↓
               │   BlockingQueue
               │
               └── Non-blocking FIFO?
                      ↓
                 ConcurrentLinkedQueue
```

---

# 30. Production Scenario — User Session Map ⭐⭐⭐⭐⭐

```java
private final ConcurrentHashMap<String, UserSession> sessions =
        new ConcurrentHashMap<>();
```

Useful when many request threads concurrently:

```text
create session
read session
update session
remove session
```

For compound updates, use APIs such as:

```java
compute()
computeIfAbsent()
computeIfPresent()
merge()
putIfAbsent()
```

rather than manually combining multiple operations without coordination.

---

# 31. Production Scenario — Event Processing Pipeline ⭐⭐⭐⭐⭐

```text
HTTP / Kafka / File Events
          ↓
       Producer
          ↓
    BlockingQueue
          ↓
      Workers
          ↓
       Database
```

A `BlockingQueue` can provide both:

```text
thread-safe handoff
+
backpressure through bounded capacity
```

---

# 32. Production Scenario — Listener Registry ⭐⭐⭐⭐

If listeners are:

```text
read frequently
registered rarely
removed rarely
```

then:

```java
CopyOnWriteArrayList<Listener>
```

may be a good fit.

If registration/removal is extremely frequent, copying the underlying array on each write may become expensive.

---

# 33. Practice — Listener Registry ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ListenerRegistry {

    private final CopyOnWriteArrayList<String> listeners =
            new CopyOnWriteArrayList<>();

    public void register(String listener) {
        listeners.addIfAbsent(listener);
    }

    public void notifyListeners() {
        for (String listener : listeners) {
            System.out.println("Notify: " + listener);
        }
    }

    public static void main(String[] args) {
        ListenerRegistry registry = new ListenerRegistry();

        registry.register("EmailListener");
        registry.register("AuditListener");

        registry.notifyListeners();
    }
}
```

---

# 34. Blocking vs Non-Blocking Queue ⭐⭐⭐⭐⭐

### Blocking

```java
queue.take();
```

If empty:

```text
consumer waits
```

### Non-blocking

```java
queue.poll();
```

If empty:

```text
returns null
```

This distinction matters in worker architecture and thread utilization.

---

# 35. `offer()` vs `put()` ⭐⭐⭐⭐⭐

For a bounded `BlockingQueue`:

```java
queue.put(item);
```

may wait until space becomes available.

While:

```java
queue.offer(item);
```

returns immediately with a success/failure result.

Timed version:

```java
queue.offer(item, 1, TimeUnit.SECONDS);
```

can wait up to the specified duration.

---

# 36. Practice — Backpressure with Bounded Queue ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BoundedQueueBackpressure {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> queue =
                new ArrayBlockingQueue<>(1);

        System.out.println(queue.offer("A")); // true
        System.out.println(queue.offer("B")); // false

        System.out.println(queue.take()); // A
        System.out.println(queue.offer("B")); // true
    }
}
```

---

# 37. Important `size()` Warning ⚠️⭐⭐⭐⭐

In concurrent collections, a value such as:

```java
queue.size()
```

should not automatically be treated as a stable synchronization fact.

Another thread may modify the collection immediately afterward.

Prefer atomic queue/map operations that directly express the required action.

---

# 38. Common Interview Traps ⭐⭐⭐⭐⭐

### Trap 1
> `ConcurrentHashMap` is just `HashMap` with one global lock.

❌ Oversimplified.

Modern implementations use more sophisticated concurrency mechanisms and internal coordination.

### Trap 2
> `CopyOnWriteArrayList` is good for frequent writes.

❌ No.

Every write can require copying the underlying array.

### Trap 3
> `BlockingQueue` always blocks.

❌ No.

Methods differ:

```text
put/take → may block
offer/poll → do not block indefinitely
```

### Trap 4
> `ConcurrentHashMap` iterator is a snapshot.

❌ No.

Its iterator is weakly consistent, not snapshot-based.

### Trap 5
> Concurrent collection makes every business operation atomic.

❌ No.

Compound operations need appropriate atomic APIs or external coordination.

### Trap 6
> `PriorityBlockingQueue` provides bounded backpressure.

❌ No.

It is unbounded.

### Trap 7
> `ConcurrentLinkedQueue` waits when empty.

❌ No.

It is non-blocking; `poll()` returns `null` when no element is available.

---

# 39. Interview Comparison Table ⭐⭐⭐⭐⭐

| Collection | Main Strength | Main Trade-Off |
|---|---|---|
| `ConcurrentHashMap` | Concurrent map access | More complex semantics than `HashMap` |
| `CopyOnWriteArrayList` | Excellent read-heavy iteration | Expensive writes |
| `ArrayBlockingQueue` | Bounded blocking queue | Fixed capacity |
| `LinkedBlockingQueue` | Flexible blocking queue | Node allocation / coordination overhead |
| `PriorityBlockingQueue` | Priority ordering | Unbounded |
| `DelayQueue` | Delayed availability | Specialized use case |
| `ConcurrentLinkedQueue` | Non-blocking FIFO | No waiting/backpressure |
| `ConcurrentSkipListMap` | Concurrent sorted map | More overhead than unordered map |
| `ConcurrentSkipListSet` | Concurrent sorted set | More overhead than simpler sets |

---

# 40. 2-Minute Interview Answer 🏆

> **"Java concurrent collections are designed for safe and efficient access from multiple threads. The main choices depend on the workload. `ConcurrentHashMap` is used for concurrent key-value access and provides atomic compound operations such as `putIfAbsent`, `compute`, and `merge`. `CopyOnWriteArrayList` is useful when reads greatly outnumber writes because writes create a new underlying array and iterators operate over a snapshot. `BlockingQueue` implementations are useful for producer-consumer systems because operations such as `put()` and `take()` can wait, making them useful for thread-safe handoff and backpressure. `ConcurrentLinkedQueue` provides a non-blocking FIFO queue when waiting is not required. `ConcurrentSkipListMap` and `ConcurrentSkipListSet` provide concurrent sorted structures. A key interview point is that thread-safe collection methods do not automatically make arbitrary multi-step business logic atomic, so we should use atomic compound APIs or explicit coordination where necessary."**

---

# 41. Quick Revision ⭐⭐⭐⭐⭐

```text
ConcurrentHashMap
→ concurrent key-value map
→ no null keys/values
→ compute/merge/putIfAbsent
→ weakly consistent iteration
```

```text
CopyOnWriteArrayList
→ read-heavy
→ write-light
→ write copies array
→ snapshot-style iterator
```

```text
BlockingQueue
→ producer-consumer
→ put/take may block
→ bounded queue can provide backpressure
```

```text
ConcurrentLinkedQueue
→ non-blocking FIFO
→ offer/poll
→ does not wait for data
```

```text
ConcurrentSkipListMap/Set
→ concurrent + sorted
```

### Golden Rules

```text
Read-heavy list      → CopyOnWriteArrayList
Concurrent map       → ConcurrentHashMap
Producer-consumer    → BlockingQueue
Non-blocking FIFO    → ConcurrentLinkedQueue
Concurrent sorted map→ ConcurrentSkipListMap
Concurrent sorted set→ ConcurrentSkipListSet
Compound map update  → compute/merge/putIfAbsent
```

---

# 42. 💻 Practice Checklist

- [ ] Create a `ConcurrentHashMap`
- [ ] Perform concurrent `put()` / `get()` operations
- [ ] Practice `putIfAbsent()`
- [ ] Practice `computeIfAbsent()`
- [ ] Practice `compute()`
- [ ] Practice `merge()`
- [ ] Understand why `ConcurrentHashMap` rejects `null`
- [ ] Explain weakly consistent iteration
- [ ] Create `CopyOnWriteArrayList`
- [ ] Practice snapshot-style iteration
- [ ] Understand read-heavy/write-light workload
- [ ] Create `ArrayBlockingQueue`
- [ ] Practice `put()` / `take()`
- [ ] Practice `offer()` / `poll()`
- [ ] Understand bounded backpressure
- [ ] Create `LinkedBlockingQueue`
- [ ] Create `PriorityBlockingQueue`
- [ ] Create `ConcurrentLinkedQueue`
- [ ] Understand blocking vs non-blocking
- [ ] Create `ConcurrentSkipListMap`
- [ ] Create `ConcurrentSkipListSet`
- [ ] Compare synchronized wrappers with concurrent collections
- [ ] Explain all major collection choices in 2 minutes

---

## Navigation

[← 8.22 — `LongAdder` / `LongAccumulator`](../22-LongAdder-and-LongAccumulator/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.24 — `ConcurrentHashMap` Deep Dive**