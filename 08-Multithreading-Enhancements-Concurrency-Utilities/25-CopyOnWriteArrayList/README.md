# 8.25 — `CopyOnWriteArrayList`

> **Goal:** Understand when and why to use `CopyOnWriteArrayList`, how copy-on-write works, iterator/snapshot semantics, write cost, thread safety, and production use cases.

---

## 1. What is `CopyOnWriteArrayList`? ⭐⭐⭐⭐⭐

`CopyOnWriteArrayList` is a thread-safe `List` implementation from `java.util.concurrent` designed primarily for workloads with **many reads and relatively few writes**.

```java
CopyOnWriteArrayList<String> list =
        new CopyOnWriteArrayList<>();
```

### Core idea

```text
Read
 ↓
Read existing internal array

Write
 ↓
Create a new array copy
 ↓
Apply modification to the new array
 ↓
Publish the new array
```

This means readers can usually read without being blocked by writers.

---

# 2. Why Does It Exist? ⭐⭐⭐⭐⭐

Consider a shared list:

```java
List<String> listeners = ...;
```

If many threads repeatedly iterate over listeners while registrations/removals are rare, locking the entire list on every read can add unnecessary contention.

`CopyOnWriteArrayList` optimizes this workload:

```text
Many reads + few writes
        ↓
CopyOnWriteArrayList
```

---

# 3. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class BasicCOWAL {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("Java");
        list.add("Spring");
        list.add("Kafka");

        System.out.println(list);
    }
}
```

---

# 4. Internal Mental Model ⭐⭐⭐⭐⭐

Conceptually, suppose the list contains:

```text
Array A
[Java, Spring, Kafka]
```

A write such as:

```java
list.add("Docker");
```

creates a new array:

```text
Array A
[Java, Spring, Kafka]
      │
      │ copy
      ▼
Array B
[Java, Spring, Kafka, Docker]
```

Then the list publishes Array B as the current array.

Existing readers using Array A can finish safely.

---

# 5. Why Is It Called Copy-On-Write? ⭐⭐⭐⭐⭐

Because the underlying array is copied **when a mutation occurs**.

```text
Read → no array copy

Write → copy array + modify copy
```

This is the central concept you must remember for interviews.

---

# 6. Read Performance ⭐⭐⭐⭐⭐

A normal read such as:

```java
String value = list.get(0);
```

reads from the current array.

Iteration can also traverse a stable array reference without coordinating with a writer for every element.

### Mental model

```text
Reader 1 ───────→ Array A
Reader 2 ───────→ Array A
Reader 3 ───────→ Array A

Writer ──copy──→ Array B
                  ↓
             publish B
```

Existing readers can continue using the old snapshot/reference.

---

# 7. Write Cost ⭐⭐⭐⭐⭐

Writes are expensive compared with reads because an array copy is required.

Examples:

```java
list.add(value);
list.remove(value);
list.set(index, value);
```

Conceptually:

```text
Write cost ≈ O(n) array copy
```

Therefore:

> `CopyOnWriteArrayList` is usually a poor choice for write-heavy workloads.

---

# 8. Practice — Concurrent Reads and Writes ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ConcurrentReadWrite {

    public static void main(String[] args) throws InterruptedException {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("Java");
        list.add("Spring");

        Thread reader = new Thread(() -> {
            for (String item : list) {
                System.out.println("Reader: " + item);
            }
        });

        Thread writer = new Thread(() -> {
            list.add("Kafka");
            list.add("Docker");
        });

        reader.start();
        writer.start();

        reader.join();
        writer.join();

        System.out.println("Final: " + list);
    }
}
```

The important point is that concurrent iteration does not require external synchronization of the entire list.

---

# 9. Iterator Snapshot Semantics ⭐⭐⭐⭐⭐

One of the most important interview concepts:

> A `CopyOnWriteArrayList` iterator traverses the array snapshot/reference that existed when the iterator was created.

Example:

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class IteratorSnapshot {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("A");
        list.add("B");

        var iterator = list.iterator();

        list.add("C");

        while (iterator.hasNext()) {
            System.out.println(iterator.next());
        }

        System.out.println("Current list: " + list);
    }
}
```

The iterator does not become a live view that automatically incorporates the later `C` insertion.

---

# 10. Practice — Snapshot Demonstration ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.Iterator;

public class SnapshotDemo {

    public static void main(String[] args) {
        CopyOnWriteArrayList<Integer> list =
                new CopyOnWriteArrayList<>();

        list.add(1);
        list.add(2);
        list.add(3);

        Iterator<Integer> iterator = list.iterator();

        list.add(4);
        list.remove(Integer.valueOf(1));

        System.out.println("Iterator sees:");
        iterator.forEachRemaining(System.out::println);

        System.out.println("Current list:");
        System.out.println(list);
    }
}
```

Expected mental model:

```text
Iterator snapshot → [1, 2, 3]
Current list      → [2, 3, 4]
```

---

# 11. Fail-Safe vs Fail-Fast ⭐⭐⭐⭐⭐

Do not use the informal phrase "fail-safe" as if it were a formal Java iterator category.

For interview purposes, remember:

```text
ArrayList iterator
→ generally fail-fast on structural modification

CopyOnWriteArrayList iterator
→ snapshot-style
→ does not throw ConcurrentModificationException because of later list modifications
```

The iterator is intentionally disconnected from later writes.

---

# 12. Can Iterator Remove Elements? ⭐⭐⭐⭐⭐

No.

For `CopyOnWriteArrayList`:

```java
iterator.remove();
```

is unsupported and throws `UnsupportedOperationException`.

### Why?

The iterator works over a stable snapshot and mutations are performed through the list itself.

---

# 13. Practice — Iterator Remove ⭐⭐⭐⭐

```java
import java.util.Iterator;
import java.util.concurrent.CopyOnWriteArrayList;

public class IteratorRemoveDemo {

    public static void main(String[] args) {
        CopyOnWriteArrayList<Integer> list =
                new CopyOnWriteArrayList<>();

        list.add(10);
        list.add(20);

        Iterator<Integer> iterator = list.iterator();

        System.out.println(iterator.next());
        iterator.remove(); // UnsupportedOperationException
    }
}
```

Run this intentionally to observe the behavior.

---

# 14. Thread Safety ⭐⭐⭐⭐⭐

`CopyOnWriteArrayList` provides thread-safe list operations.

Example:

```java
list.add("Java");
list.remove("Java");
list.set(0, "Spring");
```

These operations are coordinated internally.

But remember:

> Thread-safe collection does not automatically make arbitrary multi-step business logic atomic.

---

# 15. Compound Operation Trap ⭐⭐⭐⭐⭐

This is still a multi-step operation:

```java
if (!list.contains("Java")) {
    list.add("Java");
}
```

Two threads can both observe absence before either adds.

If the requirement is exactly-once insertion, design around an atomic operation or another synchronization strategy.

### Better options

For a set-like requirement, consider:

```java
ConcurrentHashMap.newKeySet()
```

or:

```java
Set<String> set = ConcurrentHashMap.newKeySet();
set.add("Java");
```

The data structure should match the business requirement.

---

# 16. Practice — Why List May Be the Wrong Structure ⭐⭐⭐⭐⭐

```java
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentUniqueValues {

    public static void main(String[] args) {
        Set<String> values = ConcurrentHashMap.newKeySet();

        values.add("Java");
        values.add("Java");
        values.add("Spring");

        System.out.println(values);
    }
}
```

If uniqueness is the real requirement, a concurrent set is usually a better fit than manually checking a list.

---

# 17. `addIfAbsent()` ⭐⭐⭐⭐⭐

`CopyOnWriteArrayList` provides:

```java
list.addIfAbsent("Java");
```

This performs the list's conditional insertion operation atomically according to the collection's concurrency semantics.

Example:

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class AddIfAbsentDemo {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        System.out.println(list.addIfAbsent("Java"));
        System.out.println(list.addIfAbsent("Java"));
        System.out.println(list);
    }
}
```

Expected:

```text
true
false
[Java]
```

---

# 18. `addAllAbsent()` ⭐⭐⭐⭐

You can add only elements that are not already present:

```java
list.addAllAbsent(otherList);
```

Example:

```java
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class AddAllAbsentDemo {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("Java");

        int added = list.addAllAbsent(
                List.of("Java", "Spring", "Kafka"));

        System.out.println("Added = " + added);
        System.out.println(list);
    }
}
```

---

# 19. `set()` Is Also a Write ⭐⭐⭐⭐⭐

This operation:

```java
list.set(0, "Java 21");
```

also requires copy-on-write behavior.

Do not think only `add()` and `remove()` trigger copying.

### Mutation operations include

```text
add
addAll
remove
removeAll
retainAll
set
clear
addIfAbsent
```

---

# 20. Practice — `set()` ⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class SetDemo {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("Java");
        list.add("Spring");

        list.set(0, "Java 21");

        System.out.println(list);
    }
}
```

---

# 21. Memory Cost ⭐⭐⭐⭐⭐

Every mutation may temporarily require both:

```text
old array
+
new array
```

Therefore, large lists with frequent writes can create substantial allocation/copying overhead.

### Think

```text
Read-heavy → excellent fit
Write-heavy → usually poor fit
Large frequent mutations → potentially expensive
```

---

# 22. Why Readers Don't Block Writers ⭐⭐⭐⭐⭐

The core design is based on replacing the array reference after preparing a new array.

Conceptually:

```text
Reader
  │
  └──→ old immutable-for-reader array

Writer
  │
  ├── copy old array
  ├── modify new array
  └── publish new array
```

The writer does not need to modify the array that existing readers are traversing.

---

# 23. Important: The Array Is Not Mutable In Place ⭐⭐⭐⭐⭐

Conceptually, after publication:

```text
Current array → treated as immutable for readers
```

A mutation creates a replacement array.

This is the reason snapshot iteration is possible.

---

# 24. Practice — Listener Registry ⭐⭐⭐⭐⭐

A classic production use case is a listener/subscriber registry where registration is rare but notification is frequent.

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ListenerRegistry {

    interface Listener {
        void onEvent(String event);
    }

    private final CopyOnWriteArrayList<Listener> listeners =
            new CopyOnWriteArrayList<>();

    void register(Listener listener) {
        listeners.addIfAbsent(listener);
    }

    void unregister(Listener listener) {
        listeners.remove(listener);
    }

    void publish(String event) {
        for (Listener listener : listeners) {
            listener.onEvent(event);
        }
    }

    public static void main(String[] args) {
        ListenerRegistry registry = new ListenerRegistry();

        Listener a = event -> System.out.println("A: " + event);
        Listener b = event -> System.out.println("B: " + event);

        registry.register(a);
        registry.register(b);

        registry.publish("Order Created");
    }
}
```

### Why is this a good use case?

```text
publish() → very frequent reads/iteration
register/unregister → relatively rare writes
```

---

# 25. Event Listener Example with Concurrent Registration ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ConcurrentListeners {

    private static final CopyOnWriteArrayList<String> listeners =
            new CopyOnWriteArrayList<>();

    public static void main(String[] args) throws InterruptedException {
        Thread publisher = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                for (String listener : listeners) {
                    System.out.println("Notify: " + listener);
                }
            }
        });

        Thread registrar = new Thread(() -> {
            listeners.add("Listener-A");
            listeners.add("Listener-B");
            listeners.add("Listener-C");
        });

        publisher.start();
        registrar.start();

        publisher.join();
        registrar.join();
    }
}
```

The publisher can safely iterate while registration occurs.

---

# 26. `CopyOnWriteArrayList` vs `ArrayList` ⭐⭐⭐⭐⭐

| Feature | `ArrayList` | `CopyOnWriteArrayList` |
|---|---|---|
| Thread-safe concurrent mutation | ❌ | ✅ |
| Random access | Fast | Fast |
| Write cost | Usually O(1) amortized for add-at-end | O(n) copy-oriented |
| Iterator behavior | Fail-fast in typical implementation | Snapshot-style |
| Best workload | General single-threaded use | Many reads, few writes |
| Allows null | Yes | Yes |

---

# 27. `CopyOnWriteArrayList` vs `synchronizedList()` ⭐⭐⭐⭐⭐

```java
List<String> list =
        Collections.synchronizedList(new ArrayList<>());
```

vs:

```java
CopyOnWriteArrayList<String> list =
        new CopyOnWriteArrayList<>();
```

### `synchronizedList`

Typically synchronizes individual operations through a common monitor.

Iteration requires external synchronization if you need a properly coordinated traversal:

```java
synchronized (list) {
    for (String value : list) {
        System.out.println(value);
    }
}
```

### `CopyOnWriteArrayList`

Iteration uses a stable snapshot/reference and does not require that external iteration lock.

### Interview answer

> Use `CopyOnWriteArrayList` when iteration is frequent and mutations are rare. Use a synchronized list or another concurrent structure when the workload does not fit copy-on-write semantics.

---

# 28. `CopyOnWriteArrayList` vs `ConcurrentHashMap` ⭐⭐⭐⭐⭐

Use:

```text
CopyOnWriteArrayList
→ ordered collection
→ duplicate elements allowed
→ read-heavy list workload
```

Use:

```text
ConcurrentHashMap
→ key-value lookup
→ per-key concurrent state
→ cache / counters / registry by key
```

Do not select a concurrent collection only because it is thread-safe. Select based on the required data model and workload.

---

# 29. `CopyOnWriteArrayList` vs `ConcurrentLinkedQueue` ⭐⭐⭐⭐⭐

`ConcurrentLinkedQueue` is designed for concurrent queue operations without the full-array-copy cost of copy-on-write.

```text
Need ordered read-heavy snapshot-like iteration
→ CopyOnWriteArrayList

Need frequent concurrent enqueue/dequeue
→ ConcurrentLinkedQueue / BlockingQueue depending on requirements
```

---

# 30. Does It Allow Duplicates? ⭐⭐⭐⭐⭐

Yes.

```java
CopyOnWriteArrayList<String> list =
        new CopyOnWriteArrayList<>();

list.add("Java");
list.add("Java");

System.out.println(list);
```

Output:

```text
[Java, Java]
```

`addIfAbsent()` can be used when duplicate prevention is specifically desired.

---

# 31. Does It Preserve Insertion Order? ⭐⭐⭐⭐⭐

Yes, as a `List`, it maintains index/order semantics.

```java
list.add("A");
list.add("B");
list.add("C");
```

Iteration follows list order.

Concurrent writes still happen according to the collection's thread-safe operation semantics.

---

# 32. Does It Allow `null`? ⭐⭐⭐⭐

Yes, unlike `ConcurrentHashMap`, `CopyOnWriteArrayList` permits `null` elements.

```java
list.add(null);
```

Whether using `null` is a good application design is a separate question.

---

# 33. Practice — Null and Duplicates ⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class NullAndDuplicates {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add(null);
        list.add("Java");
        list.add("Java");

        System.out.println(list);
    }
}
```

---

# 34. Atomicity vs Snapshot Consistency ⭐⭐⭐⭐⭐

A subtle distinction:

```text
Collection operation thread-safe
        ≠
whole business workflow atomic
        ≠
iterator sees latest data
```

For example, an iterator can safely traverse its snapshot while the current list has changed.

This is intentional behavior, not stale-data corruption.

---

# 35. Practice — Snapshot vs Current State ⭐⭐⭐⭐⭐

```java
import java.util.Iterator;
import java.util.concurrent.CopyOnWriteArrayList;

public class SnapshotVsCurrent {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list =
                new CopyOnWriteArrayList<>();

        list.add("A");
        list.add("B");

        Iterator<String> snapshot = list.iterator();

        list.add("C");
        list.remove("A");

        System.out.println("Snapshot:");
        snapshot.forEachRemaining(System.out::println);

        System.out.println("Current:");
        list.forEach(System.out::println);
    }
}
```

---

# 36. When NOT to Use It ⭐⭐⭐⭐⭐

Avoid `CopyOnWriteArrayList` for:

```text
frequent add/remove operations
large collections with frequent writes
write-heavy producer workloads
memory-sensitive high-mutation workloads
cases where snapshot iteration is not useful
```

Example:

```text
100,000 writes/sec
+ 10 reads/sec
```

This is generally a poor workload for copy-on-write.

---

# 37. When TO Use It ⭐⭐⭐⭐⭐

Good candidates:

```text
many reads
few writes
small/moderate collection size
iteration safety is important
readers should not block on writers
stable iteration view is useful
```

Common examples:

```text
event listeners
application hooks
configuration observers
subscriber lists
plugin registries
read-mostly metadata
```

---

# 38. Practice — Plugin Registry ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class PluginRegistry {

    private final CopyOnWriteArrayList<String> plugins =
            new CopyOnWriteArrayList<>();

    public void register(String plugin) {
        plugins.addIfAbsent(plugin);
    }

    public void unregister(String plugin) {
        plugins.remove(plugin);
    }

    public void executeAll() {
        for (String plugin : plugins) {
            System.out.println("Executing " + plugin);
        }
    }

    public static void main(String[] args) {
        PluginRegistry registry = new PluginRegistry();

        registry.register("LoggingPlugin");
        registry.register("MetricsPlugin");

        registry.executeAll();
    }
}
```

---

# 39. Memory Visibility ⭐⭐⭐⭐⭐

The collection's implementation establishes the necessary concurrency behavior so a successfully completed mutation becomes visible through subsequent accesses according to Java's concurrency semantics.

You should not add external `volatile` fields merely to make the list itself thread-safe.

The key point is:

> Use the concurrent collection's documented thread-safety guarantees instead of trying to manually reproduce its publication protocol.

---

# 40. One Writer vs Multiple Writers ⭐⭐⭐⭐

Multiple threads can mutate the list safely, but writes still involve copying and coordination.

Therefore:

```text
Thread-safe
≠
cheap under heavy write contention
```

If many threads write frequently, evaluate another concurrent collection or synchronization design.

---

# 41. Practice — Multiple Writers ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class MultipleWriters {

    public static void main(String[] args) throws InterruptedException {
        CopyOnWriteArrayList<Integer> list =
                new CopyOnWriteArrayList<>();

        Runnable writer = () -> {
            for (int i = 0; i < 1_000; i++) {
                list.add(i);
            }
        };

        Thread t1 = new Thread(writer);
        Thread t2 = new Thread(writer);
        Thread t3 = new Thread(writer);

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println("Size = " + list.size());
    }
}
```

The operations are thread-safe, but this is intentionally not an efficient workload for copy-on-write.

---

# 42. Interview Trap — "It Never Locks" ⚠️⭐⭐⭐⭐⭐

Do not say:

> `CopyOnWriteArrayList` never uses locks.

The implementation uses synchronization/locking around mutations to coordinate copying and publication.

The important property is that **read operations do not require the same kind of lock-based coordination as writes**.

### Correct answer

> Writes are coordinated and create a new backing array; reads can use the current immutable-for-reader array without blocking on every access.

---

# 43. Interview Trap — "Every Reader Gets a New Copy" ⚠️⭐⭐⭐⭐⭐

Incorrect.

The copy happens on mutation, not on every read.

```text
Reader → existing array

Writer → creates replacement array
```

---

# 44. Interview Trap — "Iterator Sees Live Updates" ⚠️⭐⭐⭐⭐⭐

Incorrect.

The iterator uses the array reference captured when it was created.

Therefore, later mutations are not automatically reflected in that iterator.

---

# 45. Interview Trap — "It Is Faster Than ArrayList" ⚠️⭐⭐⭐⭐⭐

Not generally.

For ordinary single-threaded use, `ArrayList` is usually simpler and avoids copy-on-write overhead.

`CopyOnWriteArrayList` trades expensive writes for convenient concurrent reads/iteration.

---

# 46. Performance Summary ⭐⭐⭐⭐⭐

| Operation | Mental Cost | Reason |
|---|---:|---|
| `get()` | O(1) | Array indexing |
| `contains()` | O(n) | Linear search |
| `indexOf()` | O(n) | Linear search |
| Iteration | O(n) | Traverse snapshot |
| `add()` | O(n) | Copy array |
| `set()` | O(n) | Replacement array |
| `remove()` | O(n) | Copy/shift |
| `addIfAbsent()` | O(n) | Search + copy if added |

Exact performance depends on implementation/JDK and workload, but the dominant idea is:

> **Reads are cheap; writes copy.**

---

# 47. Decision Tree 🧠

```text
Need a List?
   │
   ├── No → choose another collection
   │
   └── Yes
        │
        ├── Many reads + few writes?
        │       │
        │       └── Yes → CopyOnWriteArrayList
        │
        └── Frequent writes?
                │
                ├── Queue workload → ConcurrentLinkedQueue / BlockingQueue
                ├── Key-value → ConcurrentHashMap
                └── Other list workload → evaluate synchronization/design
```

---

# 48. Production Example — Configuration Observers ⭐⭐⭐⭐⭐

Imagine a service with configuration listeners:

```java
CopyOnWriteArrayList<Runnable> listeners =
        new CopyOnWriteArrayList<>();
```

Configuration changes may be rare:

```text
config reload → rare write
```

But every request may notify/read listeners:

```text
request processing → frequent read/iteration
```

That is exactly the kind of workload where copy-on-write can be attractive.

---

# 49. Practice — Configuration Observer ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ConfigObservers {

    private static final CopyOnWriteArrayList<Runnable> observers =
            new CopyOnWriteArrayList<>();

    public static void main(String[] args) {
        observers.add(() -> System.out.println("Refresh cache"));
        observers.add(() -> System.out.println("Refresh metrics"));

        for (Runnable observer : observers) {
            observer.run();
        }
    }
}
```

---

# 50. 2-Minute Interview Answer 🏆

> **"`CopyOnWriteArrayList` is a thread-safe List implementation designed for read-heavy, write-light workloads. Its key idea is copy-on-write: when a mutation occurs, it creates a new backing array containing the modification and publishes that array, while readers can continue using the existing array. This makes reads and iteration convenient under concurrency and gives iterators snapshot-style semantics, so they don't throw `ConcurrentModificationException` because of later list modifications. The trade-off is that writes are expensive, roughly O(n), and can create additional allocation and copying overhead. Typical use cases are event listener registries, subscriber lists, plugin registries and read-mostly configuration observers. I would not use it for write-heavy workloads; I'd consider `ConcurrentHashMap`, `ConcurrentLinkedQueue`, `BlockingQueue` or another design depending on the access pattern."**

---

# 51. Quick Revision ⭐⭐⭐⭐⭐

```text
CopyOnWriteArrayList
        ↓
Thread-safe List
        ↓
Read-heavy + write-light
        ↓
Read → current backing array
        ↓
Write → copy array
        ↓
Publish new array
        ↓
Existing iterator → snapshot/reference
        ↓
No ConcurrentModificationException from later writes
        ↓
Iterator.remove() → unsupported
        ↓
Writes ≈ O(n)
        ↓
Good → listeners / subscribers / registries
        ↓
Bad → write-heavy workloads
```

### Golden Rules

```text
Copy happens on WRITE, not READ.

Readers do not need to lock the entire list for every read.

Iterator is snapshot-style.

Thread-safe collection ≠ atomic business workflow.

Writes are expensive.

Many reads + few writes → strong candidate.

Many writes → usually choose something else.
```

---

# 52. 💻 Practice Checklist

- [ ] Create a `CopyOnWriteArrayList`
- [ ] Add/remove/set elements
- [ ] Run concurrent readers and writers
- [ ] Demonstrate iterator snapshot behavior
- [ ] Demonstrate `ConcurrentModificationException` is not thrown for later list writes
- [ ] Test `Iterator.remove()` and observe `UnsupportedOperationException`
- [ ] Practice `addIfAbsent()`
- [ ] Practice `addAllAbsent()`
- [ ] Test duplicates and `null`
- [ ] Build an event listener registry
- [ ] Build a plugin registry
- [ ] Build a configuration observer list
- [ ] Compare with `ArrayList`
- [ ] Compare with `Collections.synchronizedList()`
- [ ] Compare with `ConcurrentLinkedQueue`
- [ ] Explain copy-on-write internals
- [ ] Explain snapshot iteration
- [ ] Explain read-heavy/write-light workload
- [ ] Explain write cost and memory overhead
- [ ] Explain why it is not ideal for write-heavy workloads
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.24 — `ConcurrentHashMap` Deep Dive](../24-ConcurrentHashMap-Deep-Dive/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.26 — `BlockingQueue` Implementations**