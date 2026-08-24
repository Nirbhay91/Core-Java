# 8.37 — Parallel Streams & Concurrency Risks

> **Goal:** Understand how Java parallel streams work, when they help, when they hurt, and how to avoid common concurrency, ordering, shared-state, and thread-pool mistakes.

## 1. Mental Model ⭐⭐⭐⭐⭐

```text
Stream
  ↓
parallelStream() / parallel()
  ↓
split source
  ↓
process partitions concurrently
  ↓
combine results
```

A parallel stream is a high-level abstraction for parallel processing. It commonly uses the Fork/Join common pool, so it is important to understand the interaction between parallel streams and shared executor resources.

---

# 2. Sequential vs Parallel Stream ⭐⭐⭐⭐⭐

Sequential:

```java
list.stream()
    .map(String::toUpperCase)
    .forEach(System.out::println);
```

Parallel:

```java
list.parallelStream()
    .map(String::toUpperCase)
    .forEach(System.out::println);
```

Or:

```java
list.stream()
    .parallel()
    .map(String::toUpperCase)
    .forEach(System.out::println);
```

Interview line:

> **`parallelStream()` requests parallel stream processing; it does not guarantee that parallel execution will always be faster.**

---

# 3. Basic Parallel Stream Practice ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.stream.IntStream;

public class ParallelStreamDemo {

    public static void main(String[] args) {

        List<Integer> numbers =
                IntStream.rangeClosed(1, 20)
                        .boxed()
                        .toList();

        numbers.parallelStream()
                .forEach(number ->
                        System.out.println(
                                Thread.currentThread().getName()
                                        + " -> " + number
                        )
                );
    }
}
```

Do not expect the output order to be `1, 2, 3...20`.

---

# 4. `forEach()` vs `forEachOrdered()` ⭐⭐⭐⭐⭐

Parallel:

```java
numbers.parallelStream()
        .forEach(System.out::println);
```

Ordering is not guaranteed.

If encounter order matters:

```java
numbers.parallelStream()
        .forEachOrdered(System.out::println);
```

Important:

> `forEachOrdered()` can impose additional coordination and may reduce some of the benefit of parallel execution.

Do not say that `forEachOrdered()` makes every intermediate operation sequential; it specifically constrains the terminal consumption order.

---

# 5. Why Parallel Stream Can Be Slower ⭐⭐⭐⭐⭐

Parallelism introduces overhead:

```text
Task splitting
     ↓
Scheduling
     ↓
Thread coordination
     ↓
Combining results
```

For tiny workloads:

```java
List.of(1, 2, 3, 4).parallelStream()
```

parallel processing can easily be slower than:

```java
List.of(1, 2, 3, 4).stream()
```

Rule:

```text
Small/simple work → sequential often better
Large CPU-bound independent work → parallel may help
```

Always benchmark realistic workloads.

---

# 6. CPU-Bound vs I/O-Bound ⭐⭐⭐⭐⭐

Parallel streams are generally more naturally suited to CPU-bound, independent computations.

CPU-bound:

```java
numbers.parallelStream()
        .map(this::expensiveCpuCalculation)
        .toList();
```

I/O-bound:

```java
numbers.parallelStream()
        .map(this::callRemoteService)
        .toList();
```

The second pattern can be dangerous because blocking I/O can occupy common-pool workers.

Interview line:

> **I would not blindly use parallel streams for blocking database or HTTP calls; the shared pool can become a bottleneck and affect unrelated work.**

---

# 7. The Biggest Trap — Shared Mutable State 🚨⭐⭐⭐⭐⭐

Bad:

```java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
        .forEach(result::add);
```

`ArrayList` is not thread-safe for concurrent mutation.

Possible consequences:

```text
Race conditions
Incorrect results
Data corruption
Nondeterministic behavior
```

Better:

```java
List<Integer> result = numbers.parallelStream()
        .filter(n -> n % 2 == 0)
        .toList();
```

Prefer stream operations that return results rather than mutating external state.

---

# 8. Shared Counter Trap 🚨

Bad:

```java
int[] count = {0};

numbers.parallelStream()
        .forEach(n -> count[0]++);
```

This is not thread-safe.

Even replacing it with an atomic counter may not be the best stream design.

Prefer:

```java
long count = numbers.parallelStream()
        .filter(n -> n % 2 == 0)
        .count();
```

The reduction operation is designed to combine partial results safely.

---

# 9. Mutable Object Trap 🚨⭐⭐⭐⭐⭐

Bad:

```java
class Result {
    int total;
}

Result result = new Result();

numbers.parallelStream()
        .forEach(n -> result.total += n);
```

Multiple threads mutate the same object.

Better:

```java
int sum = numbers.parallelStream()
        .mapToInt(Integer::intValue)
        .sum();
```

Golden rule:

```text
Prefer stateless operations
        ↓
Avoid shared mutable state
        ↓
Use reduction/collectors
```

---

# 10. Associativity Matters ⭐⭐⭐⭐⭐

Parallel reduction works best when the reduction operation is associative.

Good:

```java
int sum = numbers.parallelStream()
        .reduce(0, Integer::sum);
```

Because:

```text
(a + b) + c = a + (b + c)
```

Dangerous example:

```java
int result = numbers.parallelStream()
        .reduce(0, (a, b) -> a - b);
```

Subtraction is not associative:

```text
(10 - 5) - 2 = 3
10 - (5 - 2) = 7
```

Parallel grouping can therefore produce results different from the sequential expectation.

---

# 11. Identity Must Be Correct ⭐⭐⭐⭐⭐

For:

```java
reduce(identity, accumulator)
```

the identity should be an appropriate neutral value for the operation.

Sum:

```java
.reduce(0, Integer::sum)
```

Product:

```java
.reduce(1, (a, b) -> a * b)
```

Bad identity for sum:

```java
.reduce(100, Integer::sum)
```

It changes the mathematical meaning and can interact badly with parallel reduction because partial reductions may use the identity.

---

# 12. `collect()` and Parallel Streams ⭐⭐⭐⭐⭐

Prefer collectors designed for reduction:

```java
Map<Integer, List<String>> grouped = names.parallelStream()
        .collect(Collectors.groupingBy(String::length));
```

The collector's characteristics and implementation determine how safely and efficiently the result is combined.

For highly concurrent grouping scenarios, consider:

```java
Collectors.groupingByConcurrent(...)
```

when its semantics fit the use case.

---

# 13. `groupingBy()` vs `groupingByConcurrent()` ⭐⭐⭐⭐

Normal grouping:

```java
Collectors.groupingBy(String::length)
```

Concurrent grouping:

```java
Collectors.groupingByConcurrent(String::length)
```

Do not assume `groupingByConcurrent()` is always faster. The workload, contention, number of keys, downstream collector, and data distribution matter.

---

# 14. Ordering Characteristics ⭐⭐⭐⭐⭐

A stream can have an encounter order.

For an ordered source such as a `List`:

```java
list.parallelStream()
```

some operations preserve encounter-order semantics when required.

But:

```java
forEach()
```

is explicitly nondeterministic in parallel contexts.

If ordering matters:

```java
forEachOrdered()
```

or collect into an ordered result through an appropriate operation.

---

# 15. `findFirst()` vs `findAny()` ⭐⭐⭐⭐⭐

```java
numbers.parallelStream()
        .filter(n -> n > 100)
        .findFirst();
```

`findFirst()` respects encounter order when the stream has one.

```java
numbers.parallelStream()
        .filter(n -> n > 100)
        .findAny();
```

`findAny()` is allowed to return any matching element and can be more suitable for parallel execution.

Interview line:

> **Use `findFirst()` when encounter order matters; use `findAny()` when any matching result is acceptable.**

---

# 16. `limit()` and Parallel Streams ⭐⭐⭐⭐

Operations involving order and short-circuiting can require coordination:

```java
numbers.parallelStream()
        .limit(10)
        .toList();
```

Do not assume every parallel pipeline scales linearly. Ordered stateful operations can reduce parallel efficiency.

---

# 17. Side Effects ⭐⭐⭐⭐⭐

Avoid:

```java
parallelStream()
    .map(x -> {
        log(x);
        return transform(x);
    })
```

when the side effect is shared or order-sensitive.

A side effect may be thread-safe and still make the program difficult to reason about.

Prefer:

```text
input
 ↓
stateless transformation
 ↓
reduction/collection
 ↓
result
```

---

# 18. `peek()` Is Not a Synchronization Tool 🚨

Bad idea:

```java
parallelStream()
    .peek(x -> sharedList.add(x))
    .toList();
```

`peek()` is primarily intended for debugging/observation and should not be used as a mechanism for mutating shared application state.

---

# 19. Thread Pool Interaction ⭐⭐⭐⭐⭐

Parallel streams commonly use the Fork/Join common pool.

Conceptually:

```text
parallelStream()
       ↓
Fork/Join infrastructure
       ↓
common pool
       ↓
worker threads
```

This is important in server applications because unrelated parallel-stream workloads can contend for shared pool resources.

---

# 20. Common Pool Risk in Production 🚨⭐⭐⭐⭐⭐

Imagine:

```text
Request A
   ↓
parallelStream()
   ↓
common pool

Request B
   ↓
parallelStream()
   ↓
common pool
```

Large or blocking workloads can create contention.

Interview answer:

> **I avoid using parallel streams blindly in request-processing code, especially for blocking operations, because they can consume shared common-pool workers and make resource usage less explicit.**

---

# 21. Complete Practice — Safe Parallel Sum 🏆⭐⭐⭐⭐⭐

```java
import java.util.stream.IntStream;

public class SafeParallelSum {

    public static void main(String[] args) {

        long sum = IntStream.rangeClosed(1, 1_000_000)
                .parallel()
                .asLongStream()
                .sum();

        System.out.println("Sum = " + sum);
    }
}
```

No shared counter is required.

---

# 22. Complete Practice — Safe Parallel Grouping 🏆

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class ParallelGroupingDemo {

    public static void main(String[] args) {

        List<String> names = List.of(
                "Tom", "Jerry", "John", "Alice", "Bob", "Mark"
        );

        Map<Integer, List<String>> result = names.parallelStream()
                .collect(Collectors.groupingBy(String::length));

        System.out.println(result);
    }
}
```

---

# 23. Complete Practice — `findFirst()` vs `findAny()` ⭐⭐⭐⭐⭐

```java
import java.util.List;

public class FindFirstFindAnyDemo {

    public static void main(String[] args) {

        List<Integer> numbers = List.of(
                1, 3, 5, 7, 10, 12, 14, 16
        );

        int first = numbers.parallelStream()
                .filter(n -> n > 8)
                .findFirst()
                .orElse(-1);

        int any = numbers.parallelStream()
                .filter(n -> n > 8)
                .findAny()
                .orElse(-1);

        System.out.println("First = " + first);
        System.out.println("Any = " + any);
    }
}
```

---

# 24. Complete Practice — Demonstrate Ordering ⭐⭐⭐⭐⭐

```java
import java.util.stream.IntStream;

public class OrderingDemo {

    public static void main(String[] args) {

        System.out.println("forEach:");

        IntStream.rangeClosed(1, 20)
                .parallel()
                .forEach(System.out::println);

        System.out.println("\nforEachOrdered:");

        IntStream.rangeClosed(1, 20)
                .parallel()
                .forEachOrdered(System.out::println);
    }
}
```

Observe the difference rather than assuming output order from `forEach()`.

---

# 25. Interview Coding Problem — Remove Shared State 🏆⭐⭐⭐⭐⭐

### Bad Code

```java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
        .forEach(n -> result.add(n * 2));
```

### Your Task

Rewrite it safely without explicit synchronization.

### Preferred Answer

```java
List<Integer> result = numbers.parallelStream()
        .map(n -> n * 2)
        .toList();
```

Explain:

```text
No shared mutable ArrayList
        ↓
Stateless map
        ↓
Stream collects result
```

---

# 26. Interview Coding Problem — Parallel Maximum ⭐⭐⭐⭐⭐

```java
int max = numbers.parallelStream()
        .mapToInt(Integer::intValue)
        .max()
        .orElseThrow();
```

Why is this safer than:

```java
int[] max = { Integer.MIN_VALUE };
```

Because the stream reduction handles parallel combination rather than requiring concurrent mutation of an external variable.

---

# 27. Dangerous Blocking Example 🚨

Avoid blindly doing:

```java
List<Result> results = requests.parallelStream()
        .map(request -> callRemoteService(request))
        .toList();
```

Potential issues:

```text
Blocking common-pool workers
Limited parallelism
Remote service overload
Connection pool exhaustion
Timeout amplification
Unpredictable server latency
```

For explicit I/O concurrency, an appropriately sized executor is often easier to control and reason about.

---

# 28. Parallel Stream + Synchronized Block ≠ Automatically Good 🚨

You could technically write:

```java
synchronized (lock) {
    sharedList.add(value);
}
```

But if every task contends on the same lock:

```text
parallel tasks
      ↓
     lock
      ↓
one thread at a time
```

You may destroy much of the parallelism.

Thread safety is necessary, but **thread-safe does not automatically mean scalable**.

---

# 29. Stateless vs Stateful Operations ⭐⭐⭐⭐⭐

Prefer stateless operations:

```java
map()
filter()
```

Be careful with stateful/order-sensitive operations:

```java
distinct()
sorted()
limit()
skip()
```

These can require additional coordination in parallel execution.

---

# 30. Parallel Stream Does Not Mean One Thread Per Element 🚨

Wrong:

```text
1 element = 1 thread
```

Correct mental model:

```text
Many elements
      ↓
split into chunks/tasks
      ↓
limited worker threads
      ↓
process chunks in parallel
```

Parallelism is bounded by the execution resources; it does not create an unlimited thread for every element.

---

# 31. `parallel()` Is Not a Magic Performance Switch ⭐⭐⭐⭐⭐

```java
stream.parallel()
```

changes the stream to parallel mode, but performance depends on:

```text
Source splitting
Task size
CPU cores
Operation cost
Memory locality
Ordering requirements
Contention
Collector/reduction
Data size
```

Always measure.

---

# 32. Benchmarking Rule ⭐⭐⭐⭐⭐

Do not benchmark parallel streams using:

```java
System.out.println()
```

inside the measured pipeline.

I/O can dominate the runtime.

For serious performance work, use a proper benchmarking methodology such as JMH rather than trusting one ad-hoc wall-clock run.

---

# 33. Complete Interview Scenario 🏆⭐⭐⭐⭐⭐

### Question

> You have 10 million CPU-intensive records. Would you use a parallel stream?

### Strong answer

> **"Possibly, if the operation is CPU-bound, the data set is large enough to amortize parallelization overhead, each element can be processed independently, and the reduction/collector is safe for parallel execution. I would benchmark sequential versus parallel execution under realistic production conditions. I would also consider the fact that parallel streams commonly use the Fork/Join common pool, so I would be cautious in a server application where unrelated work could contend for those workers."**

---

# 34. Complete Interview Scenario — Blocking API Calls 🏆

### Question

> You have 1,000 HTTP calls. Should you use `parallelStream()`?

### Answer

> **"Not blindly. HTTP calls are blocking I/O, so putting them into the common Fork/Join pool can occupy worker threads while they wait. I would normally prefer an explicitly managed executor or an appropriate asynchronous/non-blocking design so concurrency, queueing, timeouts and resource limits are explicit."**

---

# 35. Parallel Stream Checklist 🧠

Before using:

```text
☑ Is the workload large enough?
☑ Is it CPU-bound or appropriate for this model?
☑ Are operations independent?
☑ Is shared mutable state avoided?
☑ Is reduction associative?
☑ Is identity correct?
☑ Does ordering matter?
☑ Is blocking I/O involved?
☑ Could common-pool contention matter?
☑ Have I benchmarked it?
```

---

# 36. 2-Minute Interview Answer 🏆

> **"Parallel streams provide a high-level way to process stream elements concurrently. I can create one with `parallelStream()` or convert a stream using `parallel()`. The stream framework splits the source into tasks, processes them using available execution resources, and combines the results. I don't assume parallel is faster because task splitting, scheduling and combination have overhead. Parallel streams are generally more suitable for sufficiently large, independent CPU-bound workloads. I avoid shared mutable state such as adding to a normal `ArrayList` from `forEach`. Instead I prefer stateless operations and reductions or collectors. I also consider ordering: `forEach()` does not guarantee encounter order in parallel execution, while `forEachOrdered()` preserves encounter order at potential performance cost. Finally, parallel streams commonly use the Fork/Join common pool, so I am especially careful with blocking database or HTTP operations in server applications because they can consume shared worker resources."**

---

# 37. 30-Second Hinglish Answer

> **"Parallel Stream large aur independent workload ko multiple tasks mein process kar sakta hai. Lekin iska matlab ye nahi ki hamesha faster hoga. Task splitting aur coordination ka overhead hota hai. Shared mutable state, jaise parallel stream se normal ArrayList mein add karna, avoid karna chahiye. CPU-bound independent work mein useful ho sakta hai, lekin blocking DB/HTTP calls ke liye blindly use nahi karunga kyunki common Fork/Join pool ke workers block ho sakte hain. Ordering ke liye `forEach()` aur `forEachOrdered()` ka difference bhi important hai."**

---

# 38. Rapid-Fire Interview Questions ⭐⭐⭐⭐⭐

### Q1. How do you create a parallel stream?

```java
list.parallelStream();
```

or:

```java
list.stream().parallel();
```

### Q2. Does parallel stream guarantee faster execution?

No.

### Q3. Is `forEach()` ordered in a parallel stream?

No, not guaranteed.

### Q4. How do you preserve encounter order at the terminal operation?

```java
forEachOrdered()
```

### Q5. What is the biggest shared-state risk?

Concurrent mutation of non-thread-safe external objects.

### Q6. Why is `reduce()` preferred over a shared counter?

It supports parallel reduction without requiring shared mutable state.

### Q7. Why must reduction operations be associative?

Because partial results can be combined in different groupings.

### Q8. `findFirst()` vs `findAny()`?

`findFirst()` respects encounter order; `findAny()` can return any matching element.

### Q9. Are parallel streams good for blocking HTTP calls?

Not as a default; common-pool workers can be blocked.

### Q10. Does parallel stream create one thread per element?

No.

### Q11. What is `parallel()`?

It changes the stream to parallel mode.

### Q12. Why can `sorted()` reduce parallel efficiency?

Ordering and coordination may require additional work.

### Q13. Can a parallel stream make an unsafe object safe?

No.

### Q14. Is `synchronized` always a good fix for parallel stream state?

It can ensure safety for a critical section, but contention can destroy scalability.

### Q15. What should you do before choosing parallel?

Measure realistic workloads.

---

# 39. 🏆 Complete Interview Practice Code

Write this **without looking**:

```java
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

public class ParallelStreamInterviewPractice {

    public static void main(String[] args) {

        List<Integer> numbers = new ArrayList<>();

        for (int i = 1; i <= 100_000; i++) {
            numbers.add(i);
        }

        // 1. Parallel sum
        long sum = numbers.parallelStream()
                .mapToLong(Integer::longValue)
                .sum();

        // 2. Parallel even numbers
        List<Integer> evens = numbers.parallelStream()
                .filter(n -> n % 2 == 0)
                .toList();

        // 3. Parallel maximum
        int max = numbers.parallelStream()
                .mapToInt(Integer::intValue)
                .max()
                .orElseThrow();

        // 4. Find any matching element
        int any = numbers.parallelStream()
                .filter(n -> n > 99_000)
                .findAny()
                .orElse(-1);

        // 5. Safe grouping
        var grouped = numbers.parallelStream()
                .collect(Collectors.groupingBy(n -> n % 10));

        System.out.println("Sum = " + sum);
        System.out.println("Even count = " + evens.size());
        System.out.println("Max = " + max);
        System.out.println("Any = " + any);
        System.out.println("Groups = " + grouped.size());
    }
}
```

Then explain every line in an interview.

---

# 40. Practice Challenges 🔥

### Challenge 1 — Safe transformation

Convert every string to uppercase using a parallel stream without shared mutable state.

### Challenge 2 — Frequency grouping

Group numbers by even/odd using a parallel stream.

### Challenge 3 — Maximum

Find maximum without a shared variable.

### Challenge 4 — First vs Any

Demonstrate the difference between `findFirst()` and `findAny()`.

### Challenge 5 — Fix unsafe code

```java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
        .forEach(n -> result.add(n * 2));
```

Rewrite it safely.

### Challenge 6 — Identify production risk

```java
requests.parallelStream()
        .map(this::callDatabase)
        .toList();
```

Explain why this may be problematic.

### Challenge 7 — Explain reduction

Why is this appropriate?

```java
numbers.parallelStream()
        .reduce(0, Integer::sum);
```

---

# 41. Quick Revision 🧠

```text
parallelStream()
       ↓
parallel processing
       ↓
split + execute + combine
```

```text
Safe:
map/filter/reduce/collect
        ↓
stateless operations
        ↓
no shared mutable state
```

```text
Danger:
shared ArrayList
shared counter
shared mutable object
blocking I/O
common-pool contention
non-associative reduction
```

### Golden Rules

```text
Parallel ≠ always faster
Parallel ≠ one thread per element
forEach ≠ ordered
forEachOrdered = encounter-order constraint
findFirst = ordered semantics
findAny = any match
avoid shared mutable state
reduction should be associative
identity must be correct
benchmark before claiming performance
be careful with blocking work
```

---

# 42. Final Interview Checklist

- [ ] Explain `parallelStream()`
- [ ] Explain `stream().parallel()`
- [ ] Explain how parallel streams split work
- [ ] Explain why parallel is not always faster
- [ ] Explain `forEach()` vs `forEachOrdered()`
- [ ] Explain `findFirst()` vs `findAny()`
- [ ] Explain shared mutable state
- [ ] Fix unsafe `ArrayList` mutation
- [ ] Explain reduction and associativity
- [ ] Explain correct identity
- [ ] Explain `groupingBy()` vs `groupingByConcurrent()`
- [ ] Explain stateful operations
- [ ] Explain common-pool risk
- [ ] Explain blocking I/O risk
- [ ] Explain CPU-bound vs I/O-bound
- [ ] Write safe parallel sum
- [ ] Write safe parallel maximum
- [ ] Write grouping example
- [ ] Solve the complete interview code from memory
- [ ] Give 2-minute interview answer
- [ ] Give 30-second Hinglish answer

---

## Navigation

[← 8.36 — Recursive Tasks & Work Stealing](../36-Recursive-Tasks-and-Work-Stealing/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.38 — Thread Pool Sizing & Performance**