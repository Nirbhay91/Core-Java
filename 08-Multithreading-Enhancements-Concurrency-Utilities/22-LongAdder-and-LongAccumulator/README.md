# 8.22 — `LongAdder` / `LongAccumulator`

> **Goal:** Understand high-contention counters and accumulators, why `LongAdder` can scale better than `AtomicLong` for frequent updates, how striped cells work conceptually, and when `LongAccumulator` is appropriate.

---

## 1. Why `LongAdder`? ⭐⭐⭐⭐⭐

`LongAdder` is designed for cases where **many threads frequently update a counter** and exact value is mainly needed when reading.

Package:

```java
java.util.concurrent.atomic.LongAdder
```

Basic usage:

```java
LongAdder counter = new LongAdder();
counter.increment();
System.out.println(counter.sum());
```

### Mental Model

```text
AtomicLong
many threads
     ↓
 same atomic value
     ↓
 contention

LongAdder
many threads
     ↓
 distributed/striped cells
     ↓
 less contention in many workloads
     ↓
 sum cells when read
```

---

# 2. `LongAdder` vs `AtomicLong` ⭐⭐⭐⭐⭐

| `LongAdder` | `AtomicLong` |
|---|---|
| Optimized for high-contention updates | Good for atomic value/state operations |
| Maintains a sum across internal cells/base | Maintains one atomic value |
| `sum()` reads the current accumulated value | `get()` reads the atomic value |
| Excellent for statistics/counters | Better when exact atomic read-modify-write semantics are needed |
| Can use more memory | Simpler state representation |
| Not intended as a general replacement for `AtomicLong` | Suitable for sequence/state transitions |

### Key Interview Rule ⭐⭐⭐⭐⭐

> Use `LongAdder` when many threads update a shared counter frequently and you mainly need a running total. Use `AtomicLong` when you need atomic operations on one value, especially when the returned value participates directly in the algorithm.

---

# 3. Basic `LongAdder` Operations ⭐⭐⭐⭐⭐

Important methods:

```java
increment()
decrement()
add(long x)
sum()
sumThenReset()
reset()
```

Example:

```java
LongAdder adder = new LongAdder();

adder.increment();
adder.add(10);
adder.decrement();

System.out.println(adder.sum());
```

---

# 4. Practice — Basic `LongAdder` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAdder;

public class LongAdderBasic {

    public static void main(String[] args) {
        LongAdder counter = new LongAdder();

        counter.increment();
        counter.increment();
        counter.add(10);
        counter.decrement();

        System.out.println("Count = " + counter.sum());
    }
}
```

Expected:

```text
Count = 11
```

---

# 5. `LongAdder` Under Contention ⭐⭐⭐⭐⭐

Typical use case:

```text
Thread 1 ─┐
Thread 2 ─┤
Thread 3 ─┼──→ frequent counter updates
Thread 4 ─┤
Thread N ─┘
```

With a highly contended single atomic location, threads can repeatedly compete for updates.

`LongAdder` can spread updates across internal cells under contention and combine them when `sum()` is requested.

This can improve throughput in suitable workloads.

---

# 6. Practice — Multi-Threaded Counter ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAdder;

public class LongAdderCounter {

    public static void main(String[] args) throws InterruptedException {
        LongAdder counter = new LongAdder();

        Runnable task = () -> {
            for (int i = 0; i < 100_000; i++) {
                counter.increment();
            }
        };

        Thread[] threads = new Thread[10];

        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(task, "Worker-" + i);
            threads[i].start();
        }

        for (Thread thread : threads) {
            thread.join();
        }

        System.out.println("Final count = " + counter.sum());
    }
}
```

Expected:

```text
Final count = 1000000
```

---

# 7. How `LongAdder` Works Conceptually ⭐⭐⭐⭐⭐

The implementation has a base value and may use multiple internal cells when contention occurs.

Conceptually:

```text
             LongAdder
                 │
        ┌────────┴────────┐
        ↓                 ↓
      base             cells[]
                         │
             ┌───────────┼───────────┐
             ↓           ↓           ↓
           cell0       cell1       cell2
             ↑           ↑           ↑
           Thread A    Thread B    Thread C
```

A thread may update a cell instead of repeatedly fighting over one shared location.

> This is a conceptual model; the actual implementation details are JVM/JDK implementation-specific.

---

# 8. Why Is `LongAdder` Usually Faster Under Contention? ⭐⭐⭐⭐⭐

With many concurrent updates:

```text
AtomicLong
   ↓
one hot location
   ↓
CAS contention
   ↓
retry / cache-coherence traffic
```

`LongAdder` can instead use striped counters:

```text
Thread A → Cell 1
Thread B → Cell 2
Thread C → Cell 3
Thread D → Cell 1
```

Then:

```java
sum()
```

combines the values.

This is why `LongAdder` can have better update throughput under high contention.

---

# 9. Important: `sum()` Is Not a Snapshot Transaction ⭐⭐⭐⭐⭐

`sum()` returns the current accumulated sum, but concurrent updates can happen while the sum is being computed.

Therefore do not think:

```text
sum() = globally frozen snapshot
```

Instead think:

```text
sum()
→ current accumulated value at observation time
→ concurrent updates may occur around the read
```

This matters when exact transactional snapshots are required.

---

# 10. `sum()` vs `sumThenReset()` ⭐⭐⭐⭐⭐

### `sum()`

Returns the current sum without resetting it.

```java
LongAdder adder = new LongAdder();
adder.add(100);

long value = adder.sum();
```

The adder remains accumulated.

### `sumThenReset()`

Returns the current sum and resets the adder.

```java
long value = adder.sumThenReset();
```

Useful for periodic statistics collection.

---

# 11. Practice — Periodic Metrics ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAdder;

public class PeriodicMetrics {

    private final LongAdder requests = new LongAdder();

    public void recordRequest() {
        requests.increment();
    }

    public long collectAndReset() {
        return requests.sumThenReset();
    }

    public static void main(String[] args) {
        PeriodicMetrics metrics = new PeriodicMetrics();

        metrics.recordRequest();
        metrics.recordRequest();
        metrics.recordRequest();

        System.out.println("Requests in interval = "
                + metrics.collectAndReset());

        System.out.println("After reset = "
                + metrics.collectAndReset());
    }
}
```

---

# 12. `reset()` ⭐⭐⭐⭐

```java
LongAdder adder = new LongAdder();

adder.add(50);
adder.reset();

System.out.println(adder.sum()); // 0
```

`reset()` should be used when concurrent updates are not occurring or when the application's semantics explicitly tolerate resetting during concurrent use.

For periodic concurrent statistics, `sumThenReset()` is often the more appropriate API to consider.

---

# 13. `LongAdder` Is Not a Sequence Generator ⚠️⭐⭐⭐⭐⭐

Do not use:

```java
LongAdder
```

for:

```text
unique ID generation
sequence numbers
"give me the exact next value"
```

Why?

Because `LongAdder` is optimized for accumulation, not for an atomic "read current value and increment, returning the unique old/new sequence" operation.

Use:

```java
AtomicLong sequence = new AtomicLong();
long id = sequence.incrementAndGet();
```

for that kind of requirement.

---

# 14. Practice — `AtomicLong` for IDs ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicLong;

public class SequenceGenerator {

    private final AtomicLong sequence = new AtomicLong(1000);

    public long nextId() {
        return sequence.incrementAndGet();
    }

    public static void main(String[] args) {
        SequenceGenerator generator = new SequenceGenerator();

        System.out.println(generator.nextId());
        System.out.println(generator.nextId());
        System.out.println(generator.nextId());
    }
}
```

---

# 15. `LongAccumulator` ⭐⭐⭐⭐⭐

`LongAccumulator` is designed for repeatedly combining values using a supplied associative function.

Package:

```java
java.util.concurrent.atomic.LongAccumulator
```

Constructor:

```java
LongAccumulator accumulator =
        new LongAccumulator(Long::max, Long.MIN_VALUE);
```

The first argument is the accumulator function.

The second argument is the identity/initial value.

---

# 16. Basic `LongAccumulator` Example ⭐⭐⭐⭐⭐

Find the maximum value:

```java
import java.util.concurrent.atomic.LongAccumulator;

public class LongAccumulatorBasic {

    public static void main(String[] args) {
        LongAccumulator max =
                new LongAccumulator(Long::max, Long.MIN_VALUE);

        max.accumulate(10);
        max.accumulate(50);
        max.accumulate(20);

        System.out.println(max.get()); // 50
    }
}
```

Mental model:

```text
identity = MIN_VALUE

accumulate(10) → max(MIN, 10)
accumulate(50) → max(10, 50)
accumulate(20) → max(50, 20)

result = 50
```

---

# 17. Why Must the Accumulator Function Be Associative? ⭐⭐⭐⭐⭐

Concurrent accumulation may happen in different orders or across different internal cells.

Therefore the operation should support grouping without changing the mathematical result.

For example:

```text
(a op b) op c
```

should produce the same result as:

```text
a op (b op c)
```

for the values involved.

Good examples:

```text
addition
maximum
minimum
```

A non-associative operation can produce unexpected results when used concurrently.

---

# 18. `LongAccumulator` vs `LongAdder` ⭐⭐⭐⭐⭐

| `LongAdder` | `LongAccumulator` |
|---|---|
| Specialized for addition/counting | General-purpose accumulation function |
| `increment()` / `add()` | `accumulate(x)` |
| `sum()` | `get()` |
| Ideal for counters | Ideal for max/min/custom associative accumulation |
| Simpler | More flexible |

### Memory Trick

```text
LongAdder
→ add values
→ counter/statistics

LongAccumulator
→ combine values using a function
→ max/min/custom associative operation
```

---

# 19. Practice — Maximum Response Time ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAccumulator;

public class MaxResponseTime {

    public static void main(String[] args) {
        LongAccumulator maxResponseTime =
                new LongAccumulator(Long::max, Long.MIN_VALUE);

        maxResponseTime.accumulate(120);
        maxResponseTime.accumulate(350);
        maxResponseTime.accumulate(210);
        maxResponseTime.accumulate(90);

        System.out.println("Max = "
                + maxResponseTime.get()); // 350
    }
}
```

Production-style idea:

```text
Requests
  ↓
record latency
  ↓
LongAccumulator(max)
  ↓
maximum observed latency
```

---

# 20. Practice — Minimum Value ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAccumulator;

public class MinValueExample {

    public static void main(String[] args) {
        LongAccumulator min =
                new LongAccumulator(Long::min, Long.MAX_VALUE);

        min.accumulate(40);
        min.accumulate(15);
        min.accumulate(30);

        System.out.println("Minimum = " + min.get()); // 15
    }
}
```

---

# 21. Practice — Custom Accumulation ⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAccumulator;

public class CustomAccumulator {

    public static void main(String[] args) {
        LongAccumulator accumulator =
                new LongAccumulator((a, b) -> a + b, 0);

        accumulator.accumulate(10);
        accumulator.accumulate(20);
        accumulator.accumulate(30);

        System.out.println(accumulator.get()); // 60
    }
}
```

For simple addition, however, prefer `LongAdder`; it communicates intent more clearly and is specialized for this use case.

---

# 22. `getThenReset()` ⭐⭐⭐⭐

`LongAccumulator` provides:

```java
get()
reset()
getThenReset()
```

Example:

```java
LongAccumulator max =
        new LongAccumulator(Long::max, Long.MIN_VALUE);

max.accumulate(100);
max.accumulate(200);

long value = max.getThenReset();
```

The value is returned and the accumulator is reset to its identity value.

---

# 23. Practice — Periodic Maximum ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAccumulator;

public class PeriodicMax {

    public static void main(String[] args) {
        LongAccumulator max =
                new LongAccumulator(Long::max, Long.MIN_VALUE);

        max.accumulate(100);
        max.accumulate(400);
        max.accumulate(250);

        long intervalMax = max.getThenReset();

        System.out.println("Interval max = " + intervalMax);
        System.out.println("After reset = " + max.get());
    }
}
```

---

# 24. `LongAccumulator` Identity Value ⭐⭐⭐⭐⭐

The identity value must be chosen carefully.

For maximum:

```java
new LongAccumulator(Long::max, Long.MIN_VALUE)
```

For minimum:

```java
new LongAccumulator(Long::min, Long.MAX_VALUE)
```

For addition:

```java
new LongAccumulator((a, b) -> a + b, 0)
```

Wrong identity values can produce incorrect results.

---

# 25. `LongAdder` vs `LongAccumulator` vs `AtomicLong` ⭐⭐⭐⭐⭐

| Requirement | Recommended tool |
|---|---|
| High-contention counter | `LongAdder` |
| High-contention addition | `LongAdder` |
| Max/min/custom associative accumulation | `LongAccumulator` |
| Exact atomic value / sequence | `AtomicLong` |
| CAS-based state transition | `AtomicLong` / suitable atomic class |

This is a common interview comparison.

---

# 26. `LongAdder` vs `synchronized` ⭐⭐⭐⭐⭐

Traditional counter:

```java
synchronized void increment() {
    count++;
}
```

High-contention statistics counter:

```java
LongAdder counter = new LongAdder();
counter.increment();
```

`LongAdder` can avoid a single monitor bottleneck for suitable counter workloads.

But it is **not** a universal replacement for synchronization.

If an operation needs:

```text
multiple fields
+ validation
+ invariant
+ state transition
```

then a lock or another coordinated design may still be required.

---

# 27. Practice — Request Metrics ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.LongAdder;

public class RequestMetrics {

    private final LongAdder successful = new LongAdder();
    private final LongAdder failed = new LongAdder();

    public void recordSuccess() {
        successful.increment();
    }

    public void recordFailure() {
        failed.increment();
    }

    public void printMetrics() {
        System.out.println("Success = " + successful.sum());
        System.out.println("Failure = " + failed.sum());
    }

    public static void main(String[] args) {
        RequestMetrics metrics = new RequestMetrics();

        metrics.recordSuccess();
        metrics.recordSuccess();
        metrics.recordFailure();

        metrics.printMetrics();
    }
}
```

This is a good real-world example of where `LongAdder` fits naturally.

---

# 28. `LongAdder` and Exact Reads ⚠️⭐⭐⭐⭐⭐

A common interview trap:

> "Is `LongAdder.sum()` an atomic snapshot that can be used as an exact synchronization decision?"

Do not assume that.

`LongAdder` is designed for scalable accumulation. Its read is not intended to provide a transactionally frozen global state while other threads are updating it.

If your algorithm needs a precise atomic read-modify-write decision on one value, `AtomicLong` may be the better abstraction.

---

# 29. Why `LongAdder` Is Not Ideal for `if (sum() < limit) add()` ⚠️⭐⭐⭐⭐⭐

This is unsafe as a global limit mechanism:

```java
if (counter.sum() < limit) {
    counter.increment();
}
```

Two threads can both observe a value below the limit and both increment.

If the limit must be enforced atomically, use an abstraction that supports the required atomic state transition, such as a CAS loop with `AtomicLong` or an appropriate lock.

---

# 30. Practice — Atomic Limit with `AtomicLong` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.atomic.AtomicLong;

public class AtomicLimit {

    private final AtomicLong counter = new AtomicLong();
    private final long limit;

    public AtomicLimit(long limit) {
        this.limit = limit;
    }

    public boolean tryIncrement() {
        while (true) {
            long current = counter.get();

            if (current >= limit) {
                return false;
            }

            if (counter.compareAndSet(current, current + 1)) {
                return true;
            }
        }
    }

    public long get() {
        return counter.get();
    }

    public static void main(String[] args) {
        AtomicLimit limit = new AtomicLimit(2);

        System.out.println(limit.tryIncrement()); // true
        System.out.println(limit.tryIncrement()); // true
        System.out.println(limit.tryIncrement()); // false
        System.out.println(limit.get());           // 2
    }
}
```

---

# 31. Performance Benchmark Warning ⚠️⭐⭐⭐⭐⭐

Do not claim:

> "`LongAdder` is always faster than `AtomicLong`."

Correct answer:

> `LongAdder` is designed to improve scalability under contention for accumulation workloads. Actual performance depends on contention, hardware, workload, read frequency, and JVM implementation.

Use a proper benchmark such as **JMH** when performance matters.

---

# 32. When NOT to Use `LongAdder` ⭐⭐⭐⭐⭐

Avoid it when you need:

- unique sequential IDs
- exact atomic compare-and-set state transitions
- a value that directly participates in an atomic decision
- multi-variable transactional invariants
- a simple low-contention atomic value where `AtomicLong` is clearer

Use the abstraction that matches the operation's semantics.

---

# 33. Production Scenario — Requests Per Second ⭐⭐⭐⭐⭐

A web service can maintain a high-throughput request counter:

```java
private final LongAdder requestCount = new LongAdder();

public void recordRequest() {
    requestCount.increment();
}

public long totalRequests() {
    return requestCount.sum();
}
```

A monitoring thread can periodically read the counter and calculate rates.

---

# 34. Production Scenario — Maximum Latency ⭐⭐⭐⭐⭐

```java
private final LongAccumulator maxLatency =
        new LongAccumulator(Long::max, Long.MIN_VALUE);

public void recordLatency(long latencyMillis) {
    maxLatency.accumulate(latencyMillis);
}
```

Then:

```java
long max = maxLatency.get();
```

This is a natural `LongAccumulator` use case.

---

# 35. Production Scenario — Periodic Metrics Collection ⭐⭐⭐⭐⭐

```text
Application threads
       ↓
LongAdder / LongAccumulator
       ↓
metrics collector
       ↓
sumThenReset() / getThenReset()
       ↓
interval statistics
```

This pattern is useful when you care about **per-interval statistics** rather than one permanent counter.

---

# 36. Important Memory Model Point ⭐⭐⭐⭐

`LongAdder` and `LongAccumulator` provide thread-safe accumulation operations, but their semantics are intentionally different from using one atomic variable for every decision.

The key is to understand the abstraction:

```text
LongAdder
→ scalable accumulation

LongAccumulator
→ scalable associative accumulation

AtomicLong
→ atomic single-value operations
```

---

# 37. Common Mistake — Using `sumThenReset()` as a Perfect Transaction ⚠️

Do not assume that:

```java
long x = adder.sumThenReset();
```

creates a globally frozen transaction boundary where every concurrent update is perfectly classified before or after the interval.

Concurrent updates around collection/reset require careful metric semantics.

For production metrics, define what the interval means and tolerate the intended concurrent semantics.

---

# 38. Common Mistake — Using a Non-Associative Accumulator ⚠️⭐⭐⭐⭐⭐

Avoid using operations whose result depends on grouping/order when concurrent accumulation may reorder/group operations.

For example, subtraction is not associative:

```text
(a - b) - c
```

is generally different from:

```text
a - (b - c)
```

Prefer associative functions for `LongAccumulator`.

---

# 39. Practice — Compare Three Counters ⭐⭐⭐⭐⭐

Implement the same workload with:

```text
1. synchronized counter
2. AtomicLong
3. LongAdder
```

Example structure:

```java
for (int i = 0; i < 100_000; i++) {
    counter.increment();
}
```

Then run with multiple threads and compare throughput using a proper benchmark framework for meaningful performance conclusions.

---

# 40. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `LongAdder`?

`LongAdder` is a high-throughput concurrent accumulator designed primarily for counters that receive frequent updates from many threads.

### Q2. Why can `LongAdder` outperform `AtomicLong`?

Under contention, it can distribute updates across internal cells instead of forcing every thread to update one hot atomic location.

### Q3. Is `LongAdder` always faster?

No. It is optimized for particular high-contention accumulation workloads. Actual performance depends on workload and environment.

### Q4. Why use `sum()` instead of `get()`?

`LongAdder` maintains a sum across its internal state, so `sum()` returns the accumulated value.

### Q5. Can `LongAdder` generate unique IDs?

It is not the right abstraction for that requirement. Use `AtomicLong` for an atomic sequence operation.

### Q6. What is `LongAccumulator`?

It is a concurrent accumulator that combines values using a supplied associative function.

### Q7. Give examples of `LongAccumulator`.

Maximum, minimum, and other associative accumulation functions.

### Q8. Why must the accumulator function be associative?

Because concurrent accumulation may combine partial results in different groupings, and associativity allows those groupings to produce the same result.

### Q9. What is the difference between `sum()` and `sumThenReset()`?

`sum()` reads the current accumulated value; `sumThenReset()` returns the accumulated value and resets the adder.

### Q10. `LongAdder` vs `AtomicLong`?

Use `LongAdder` for high-contention statistics/counters; use `AtomicLong` when you need atomic single-value operations such as CAS or sequence generation.

### Q11. Is `LongAdder.sum()` a transactional snapshot?

No. It should not be treated as a globally frozen snapshot while concurrent updates are occurring.

### Q12. Can `LongAccumulator` use subtraction safely?

Generally not for a parallel accumulation algorithm because subtraction is not associative.

---

# 41. 2-Minute Interview Answer 🏆

> **"`LongAdder` is a class from `java.util.concurrent.atomic` designed for high-contention counters. Unlike `AtomicLong`, which operates on a single atomic value, `LongAdder` can distribute updates across internal cells under contention and combine them when `sum()` is called. This can provide better update throughput for statistics and counters when many threads are incrementing frequently. It should not be used for unique ID generation or exact atomic decisions such as `if (value < limit) increment`, because `sum()` is not a transactional snapshot. For those cases, `AtomicLong` with CAS may be more appropriate. `LongAccumulator` is a more general version for associative accumulation functions such as maximum or minimum. Its operation must be associative because concurrent updates may be combined in different groupings. In short: `LongAdder` is for scalable addition, `LongAccumulator` is for scalable associative accumulation, and `AtomicLong` is for atomic operations on one value."**

---

# 42. Quick Revision ⭐⭐⭐⭐⭐

```text
LongAdder
   ↓
high-contention counter
   ↓
striped/internal cells
   ↓
less contention for updates
   ↓
sum() when reading
```

```text
LongAccumulator
      ↓
associative function
      ↓
max / min / custom accumulation
      ↓
get()
```

### Golden Rules

```text
LongAdder → high-contention addition/counters
AtomicLong → exact atomic single-value operations
LongAccumulator → associative custom accumulation
sum() → not a transactional global snapshot
sumThenReset() → read + reset semantics
LongAdder → not for unique IDs
CAS limit → AtomicLong / lock, not sum()+increment()
Associative function → essential for LongAccumulator
Performance claims → benchmark with JMH
```

---

# 43. 💻 Practice Checklist

- [ ] Create `LongAdder`
- [ ] Use `increment()`
- [ ] Use `decrement()`
- [ ] Use `add()`
- [ ] Use `sum()`
- [ ] Use `reset()`
- [ ] Use `sumThenReset()`
- [ ] Build a multi-threaded counter
- [ ] Understand contention and striping conceptually
- [ ] Compare `LongAdder` vs `AtomicLong`
- [ ] Understand why `LongAdder` is not an ID generator
- [ ] Create `LongAccumulator`
- [ ] Use `Long::max`
- [ ] Use `Long::min`
- [ ] Create a custom associative accumulator
- [ ] Use `get()`
- [ ] Use `getThenReset()`
- [ ] Choose correct identity values
- [ ] Explain associativity
- [ ] Explain `sum()` semantics under concurrent updates
- [ ] Identify when `AtomicLong` is required instead
- [ ] Explain all three in 2 minutes

---

## Navigation

[← 8.21 — Atomic Variables & CAS](../21-Atomic-Variables-and-CAS/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.23 — Concurrent Collections Overview**