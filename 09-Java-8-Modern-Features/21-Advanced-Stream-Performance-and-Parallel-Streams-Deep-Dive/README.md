# 9.21 — Advanced Stream Performance & Parallel Streams

## 🎯 Interview Goal

Understand when Streams are efficient, when `parallelStream()` helps, when it hurts, and how to reason about performance in a 5-year Java interview.

> **Important:** Streams are primarily a readability/functional abstraction. Do not claim that streams are automatically faster than loops, or that parallel streams automatically improve performance.

---

# 1. Sequential Stream vs Parallel Stream ⭐⭐⭐⭐⭐

Sequential:

```java
long count = employees.stream()
        .filter(e -> e.getSalary() >= 100000)
        .count();
```

Parallel:

```java
long count = employees.parallelStream()
        .filter(e -> e.getSalary() >= 100000)
        .count();
```

The key difference is how the stream pipeline is executed: a parallel stream may split work and process partitions concurrently using the common `ForkJoinPool`.

---

# 2. What Actually Happens in a Parallel Stream? 🔥🔥🔥

Mental model:

```text
Source
  ↓
Spliterator
  ↓
Split into partitions
  ↓
Fork/Join tasks
  ↓
Parallel processing
  ↓
Combine partial results
```

A parallel stream does not simply mean:

```text
one thread per element ❌
```

It uses task partitioning and fork/join execution.

---

# 3. `Spliterator` and Splitting ⭐⭐⭐⭐⭐

Parallel streams depend heavily on the source's ability to split efficiently.

```java
List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
        .boxed()
        .collect(Collectors.toList());

long sum = numbers.parallelStream()
        .mapToLong(Integer::longValue)
        .sum();
```

Array/list-like sources generally split efficiently.

Interview line:

> A parallel stream works best when its source can be split into reasonably balanced chunks with low splitting overhead.

---

# 4. CPU-Bound vs I/O-Bound Work 🔥🔥🔥

### CPU-bound

Examples:

```text
complex calculation
image processing
large numerical transformation
CPU-heavy parsing
```

Parallel streams may help when:

```text
large dataset
+ expensive CPU work
+ enough CPU cores
+ low coordination overhead
```

### I/O-bound

Examples:

```text
HTTP calls
Database calls
File/network waits
```

Do not blindly use `parallelStream()` for I/O.

It uses the common `ForkJoinPool`, and blocking operations can consume worker threads and interfere with unrelated common-pool tasks.

---

# 5. Small Dataset: Parallel Can Be Slower

Parallel execution has overhead:

```text
partitioning
scheduling
thread coordination
merging results
```

For small work:

```java
numbers.stream()
```

can easily beat:

```java
numbers.parallelStream()
```

Never decide from theory alone for a production workload—benchmark representative data.

---

# 6. `forEach()` Ordering Difference 🔥

Sequential:

```java
numbers.stream()
        .forEach(System.out::println);
```

Parallel:

```java
numbers.parallelStream()
        .forEach(System.out::println);
```

Parallel `forEach()` does **not guarantee encounter order**.

If encounter order matters:

```java
numbers.parallelStream()
        .forEachOrdered(System.out::println);
```

But `forEachOrdered()` can reduce the benefits of parallel execution because ordering constraints must be respected.

---

# 7. Stateful Operations Can Be Expensive ⚠️

Operations such as:

```text
sorted()
distinct()
limit()
skip()
```

may require coordination across partitions.

Example:

```java
numbers.parallelStream()
        .sorted()
        .limit(100)
        .collect(Collectors.toList());
```

Do not assume every intermediate operation parallelizes equally well.

---

# 8. Stateless vs Stateful Operations 🔥

### Stateless

Each element can generally be processed independently:

```java
filter()
map()
mapToInt()
peek()
```

### Stateful

May need information from other elements/partitions:

```java
sorted()
distinct()
limit()
skip()
```

Interview line:

> Stateless operations generally scale more naturally in parallel execution, while stateful operations can require buffering or coordination and therefore may reduce parallel efficiency.

---

# 9. Avoid Shared Mutable State ❌🔥

Bad:

```java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
        .forEach(result::add);
```

`ArrayList` is not designed for concurrent mutation like this.

Better:

```java
List<Integer> result = numbers.parallelStream()
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());
```

Let the stream framework manage accumulation and combination.

---

# 10. Side Effects Are Dangerous in Parallel Streams

Avoid:

```java
AtomicInteger counter = new AtomicInteger();

numbers.parallelStream()
        .forEach(n -> counter.incrementAndGet());
```

This may be thread-safe, but it is usually unnecessary.

Prefer:

```java
long counter = numbers.parallelStream()
        .count();
```

Thread-safe does not automatically mean good stream design.

---

# 11. Non-Interference 🔥🔥🔥

A stream pipeline should not modify the source while it is being processed.

Bad:

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4));

numbers.stream()
        .forEach(n -> numbers.add(n * 2));
```

This can cause concurrent modification problems and violates the intended non-interfering usage model.

---

# 12. Associativity and `reduce()` 🔥🔥🔥

Parallel reduction requires an operation that can be safely combined across partitions.

Good:

```java
int sum = numbers.parallelStream()
        .reduce(0, Integer::sum);
```

Sum is associative:

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
(a - b) - c != a - (b - c)
```

Therefore parallel reduction can produce unexpected results.

---

# 13. Identity Must Be Correct 🔥🔥🔥

For reduction:

```java
stream.reduce(identity, accumulator)
```

The identity must be appropriate for the operation.

For sum:

```java
reduce(0, Integer::sum)
```

For multiplication:

```java
reduce(1, (a, b) -> a * b)
```

Do not use an arbitrary non-identity initial value in a parallel reduction.

---

# 14. `collect()` and Parallel Streams

Collectors can support parallel accumulation when their characteristics and implementation permit it.

Example:

```java
Map<String, List<Employee>> result = employees.parallelStream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment
        ));
```

The framework can accumulate partial results and combine them.

---

# 15. `groupingBy()` vs `groupingByConcurrent()` ⭐⭐⭐⭐⭐

Sequential/common grouping:

```java
Map<String, List<Employee>> result = employees.parallelStream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Concurrent grouping collector:

```java
ConcurrentMap<String, List<Employee>> result = employees.parallelStream()
        .collect(Collectors.groupingByConcurrent(Employee::getDepartment));
```

`groupingByConcurrent()` is designed for concurrent accumulation into a `ConcurrentMap` and can be advantageous with parallel streams for suitable workloads.

Do not assume it is always faster.

---

# 16. `toConcurrentMap()` 🔥

```java
ConcurrentMap<Integer, Employee> result = employees.parallelStream()
        .collect(Collectors.toConcurrentMap(
                Employee::getId,
                Function.identity(),
                (e1, e2) -> e1
        ));
```

You need a merge function if duplicate keys are possible.

---

# 17. `unordered()` Can Help

If encounter order is not semantically important:

```java
long count = numbers.parallelStream()
        .unordered()
        .filter(n -> n % 2 == 0)
        .count();
```

Removing unnecessary ordering constraints can improve parallel execution in appropriate pipelines.

Only use it when order truly does not matter.

---

# 18. `findFirst()` vs `findAny()` 🔥🔥🔥

```java
Optional<Integer> first = numbers.parallelStream()
        .filter(n -> n > 100)
        .findFirst();
```

`findFirst()` respects encounter order when an encounter order exists.

```java
Optional<Integer> any = numbers.parallelStream()
        .filter(n -> n > 100)
        .findAny();
```

`findAny()` is free to return any matching element and can be more suitable for parallel execution when the exact first element is irrelevant.

Interview line:

> If I only need any matching result, `findAny()` can avoid the ordering requirement that `findFirst()` has.

---

# 19. Common Pool 🔥🔥🔥

A parallel stream normally uses the common `ForkJoinPool`.

Conceptually:

```java
ForkJoinPool.commonPool()
```

Example:

```java
System.out.println(ForkJoinPool.commonPool().getParallelism());
```

Do not confuse stream parallelism with creating a new thread pool for every stream.

---

# 20. Parallel Stream + Blocking I/O ⚠️

Avoid designs like:

```java
urls.parallelStream()
        .map(this::callRemoteService)
        .collect(Collectors.toList());
```

when the remote calls can block for significant time and the common pool is shared with other application work.

For controlled I/O concurrency, prefer an explicitly designed executor / `CompletableFuture` approach where appropriate.

---

# 21. Benchmarking Streams Correctly ⭐⭐⭐⭐⭐

Do not benchmark like this:

```java
long start = System.nanoTime();
// one tiny operation
long end = System.nanoTime();
```

and conclude that one approach is always faster.

For serious JVM benchmarking, use **JMH (Java Microbenchmark Harness)**.

Benchmark:

```text
same input
same work
multiple iterations
warmup
measurement
forks
representative data size
```

Interview line:

> I would use JMH rather than a single `System.nanoTime()` measurement for a reliable JVM microbenchmark.

---

# 22. Example Benchmark Target

Compare:

```java
long sequential = numbers.stream()
        .mapToLong(this::expensiveCalculation)
        .sum();
```

vs:

```java
long parallel = numbers.parallelStream()
        .mapToLong(this::expensiveCalculation)
        .sum();
```

Measure on realistic data and workload.

---

# 23. When Parallel Streams Are Good ⭐⭐⭐⭐⭐

A strong candidate usually has:

```text
✓ large enough dataset
✓ expensive CPU-bound operation
✓ independent elements
✓ low synchronization
✓ efficient splittable source
✓ associative reduction/combination
✓ enough available CPU
✓ no important ordering requirement
```

---

# 24. When Parallel Streams Are Bad ❌

Common warning signs:

```text
✗ tiny dataset
✗ cheap per-element operation
✗ heavy synchronization
✗ shared mutable state
✗ blocking I/O
✗ strict ordering requirement
✗ expensive splitting/combining
✗ workload already saturates CPU
✗ common-pool contention
```

---

# 25. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.Function;
import java.util.stream.*;

public class AdvancedStreamPerformanceDemo {

    static class Employee {
        private final int id;
        private final String name;
        private final String department;
        private final int salary;

        Employee(int id, String name, String department, int salary) {
            this.id = id;
            this.name = name;
            this.department = department;
            this.salary = salary;
        }

        public int getId() {
            return id;
        }

        public String getName() {
            return name;
        }

        public String getDepartment() {
            return department;
        }

        public int getSalary() {
            return salary;
        }

        @Override
        public String toString() {
            return id + " - " + name + " - " + department + " - " + salary;
        }
    }

    public static void main(String[] args) {

        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
                .boxed()
                .collect(Collectors.toList());

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000),
                new Employee(2, "Rahul", "IT", 120000),
                new Employee(3, "Amit", "HR", 90000),
                new Employee(4, "Priya", "HR", 130000),
                new Employee(5, "Sneha", "Finance", 110000)
        );

        // 1. Sequential stream
        long sequentialCount = numbers.stream()
                .filter(n -> n % 2 == 0)
                .count();

        // 2. Parallel stream
        long parallelCount = numbers.parallelStream()
                .filter(n -> n % 2 == 0)
                .count();

        // 3. Parallel sum with associative reduction
        long parallelSum = numbers.parallelStream()
                .mapToLong(Integer::longValue)
                .sum();

        // 4. forEach: order is not guaranteed in parallel
        numbers.parallelStream()
                .limit(10)
                .forEach(n -> System.out.print(n + " "));
        System.out.println();

        // 5. forEachOrdered: preserves encounter order
        numbers.parallelStream()
                .limit(10)
                .forEachOrdered(n -> System.out.print(n + " "));
        System.out.println();

        // 6. findFirst vs findAny
        Optional<Integer> first = numbers.parallelStream()
                .filter(n -> n > 900_000)
                .findFirst();

        Optional<Integer> any = numbers.parallelStream()
                .filter(n -> n > 900_000)
                .findAny();

        // 7. groupingBy with parallel stream
        Map<String, List<Employee>> grouped = employees.parallelStream()
                .collect(Collectors.groupingBy(Employee::getDepartment));

        // 8. groupingByConcurrent
        ConcurrentMap<String, List<Employee>> concurrentGrouped = employees.parallelStream()
                .collect(Collectors.groupingByConcurrent(Employee::getDepartment));

        // 9. toConcurrentMap
        ConcurrentMap<Integer, Employee> employeeMap = employees.parallelStream()
                .collect(Collectors.toConcurrentMap(
                        Employee::getId,
                        Function.identity(),
                        (e1, e2) -> e1
                ));

        // 10. unordered when order does not matter
        long unorderedCount = numbers.parallelStream()
                .unordered()
                .filter(n -> n % 2 == 0)
                .count();

        // 11. Safe reduction
        int safeSum = IntStream.rangeClosed(1, 100)
                .parallel()
                .reduce(0, Integer::sum);

        // 12. Demonstrate common pool
        ForkJoinPool pool = ForkJoinPool.commonPool();
        System.out.println("Common pool parallelism: " + pool.getParallelism());

        // 13. Thread observation
        numbers.parallelStream()
                .limit(20)
                .forEach(n -> System.out.println(
                        "n=" + n + ", thread=" + Thread.currentThread().getName()));

        // 14. CPU-style calculation
        long cpuResult = numbers.parallelStream()
                .mapToLong(AdvancedStreamPerformanceDemo::expensiveCalculation)
                .sum();

        System.out.println("Sequential count: " + sequentialCount);
        System.out.println("Parallel count: " + parallelCount);
        System.out.println("Parallel sum: " + parallelSum);
        System.out.println("findFirst: " + first);
        System.out.println("findAny: " + any);
        System.out.println("Grouped: " + grouped);
        System.out.println("Concurrent grouped: " + concurrentGrouped);
        System.out.println("Concurrent map: " + employeeMap);
        System.out.println("Unordered count: " + unorderedCount);
        System.out.println("Safe sum: " + safeSum);
        System.out.println("CPU result: " + cpuResult);
    }

    private static long expensiveCalculation(int n) {
        long result = n;
        for (int i = 0; i < 100; i++) {
            result = (result * 31 + i) ^ (result >>> 3);
        }
        return result;
    }
}
```

---

# 26. Interview Scenario — Should I Use `parallelStream()`? 🔥🔥🔥

**Question:** You have 10 million records and each record requires expensive CPU computation. Would you use `parallelStream()`?

### Strong answer

> “Possibly, but I would not decide only from the record count. I would check whether the work is CPU-bound and independent, whether the source splits efficiently, whether the reduction is associative, whether ordering is required, and whether the application is already using the common ForkJoinPool heavily. Then I would benchmark a realistic workload. If the operation is blocking I/O, I would generally avoid using the common-pool parallel stream as the default concurrency mechanism.”

---

# 27. Interview Scenario — Why Is Parallel Slower?

Possible reasons:

```text
small dataset
cheap operation
splitting overhead
combination overhead
synchronization
ordering constraints
stateful operations
common-pool contention
insufficient CPU work per element
```

Do not answer:

> “Parallel streams are always slower.” ❌

Correct:

> “Parallel streams have overhead, so they help only when the workload is large and sufficiently expensive to amortize that overhead.”

---

# 28. Interview Scenario — Is This Thread-Safe?

```java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
        .forEach(result::add);
```

Answer:

> “No. `ArrayList` is not safe for concurrent structural modification. More importantly, I would avoid this side-effecting pattern and use a collector.”

Better:

```java
List<Integer> result = numbers.parallelStream()
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());
```

---

# 29. Interview Scenario — `findFirst()` vs `findAny()`

```java
numbers.parallelStream()
        .filter(n -> n > 100)
        .findAny();
```

Use `findAny()` when any matching element is acceptable.

Use:

```java
findFirst()
```

when encounter-order semantics matter.

---

# 30. Interview Scenario — `groupingBy()` vs `groupingByConcurrent()`

```text
Collectors.groupingBy()
→ normal Map result
→ combines partial results

Collectors.groupingByConcurrent()
→ ConcurrentMap result
→ designed for concurrent grouping
```

Interview answer:

> “`groupingByConcurrent()` can reduce combination overhead in suitable parallel workloads, but it is not automatically faster. I would benchmark with the actual data distribution and downstream collector.”

---

# 31. Interview Scenario — Parallel `reduce()`

Bad conceptual choice:

```java
numbers.parallelStream()
        .reduce(0, (a, b) -> a - b);
```

Why?

```text
subtraction is not associative
```

Good:

```java
numbers.parallelStream()
        .reduce(0, Integer::sum);
```

Because addition is associative for integer arithmetic in this context.

---

# 32. 25 Interview Questions 🎯

1. How does a parallel stream work?
2. What is the role of `Spliterator`?
3. Why is a splittable source important?
4. Does parallel stream create one thread per element?
5. Which pool does a parallel stream normally use?
6. What is the common `ForkJoinPool`?
7. When should you use parallel streams?
8. When should you avoid them?
9. Why can a parallel stream be slower?
10. CPU-bound vs I/O-bound parallel stream?
11. Why is blocking I/O dangerous in the common pool?
12. Difference between `forEach()` and `forEachOrdered()`?
13. Difference between `findFirst()` and `findAny()`?
14. What are stateful stream operations?
15. Why can `sorted()` reduce parallel performance?
16. What is a non-interfering stream operation?
17. Why is shared mutable state dangerous?
18. What makes a `reduce()` operation suitable for parallel execution?
19. Why must the identity be valid?
20. `groupingBy()` vs `groupingByConcurrent()`?
21. What does `unordered()` do?
22. Does `parallelStream()` guarantee speedup?
23. How would you benchmark streams?
24. Why is JMH preferred for microbenchmarks?
25. Would you use `parallelStream()` for database/API calls? Explain.

---

# 33. 25 Coding Challenges 💻

### Challenge 1
Compare sequential and parallel counting for 1 million integers.

### Challenge 2
Calculate a large CPU-bound sum sequentially and in parallel.

### Challenge 3
Print thread names used by a parallel stream.

### Challenge 4
Demonstrate `forEach()` ordering behavior.

### Challenge 5
Demonstrate `forEachOrdered()`.

### Challenge 6
Compare `findFirst()` and `findAny()`.

### Challenge 7
Find maximum salary using a parallel stream.

### Challenge 8
Group employees using `groupingBy()` on a parallel stream.

### Challenge 9
Group employees using `groupingByConcurrent()`.

### Challenge 10
Create a concurrent employee map using `toConcurrentMap()`.

### Challenge 11
Show why modifying an `ArrayList` from `parallelStream().forEach()` is unsafe.

### Challenge 12
Rewrite the previous challenge using `collect()`.

### Challenge 13
Create a parallel reduction using addition.

### Challenge 14
Demonstrate why subtraction is unsuitable for a parallel reduction.

### Challenge 15
Use `unordered()` when encounter order is irrelevant.

### Challenge 16
Compare `sorted()` performance conceptually in sequential vs parallel streams.

### Challenge 17
Calculate filtered salary statistics using a parallel stream.

### Challenge 18
Build a department summary using `groupingByConcurrent()`.

### Challenge 19
Create an expensive CPU calculation and compare sequential vs parallel execution.

### Challenge 20
Write a JMH benchmark for sequential vs parallel calculation.

### Challenge 21
Demonstrate the effect of a tiny dataset on parallel overhead.

### Challenge 22
Demonstrate common-pool contention.

### Challenge 23
Explain why blocking operations should not blindly run in a parallel stream.

### Challenge 24
Identify which operations in a stream pipeline are stateless vs stateful.

### Challenge 25 — 5-Year Interview Level 🔥🔥🔥
Given a production workload, decide between:

```text
for-loop
sequential stream
parallel stream
CompletableFuture + custom executor
```

Explain the choice based on CPU/I/O nature, workload size, ordering, isolation, resource limits, and benchmark results.

---

# 34. Common Mistakes ❌

### ❌ “Parallel stream is always faster.”

False. Parallelism has overhead.

### ❌ “One element gets one thread.”

False. Work is partitioned into tasks.

### ❌ “Parallel streams are ideal for HTTP calls.”

Not as a blanket rule. Blocking I/O can exhaust or contend for common-pool workers.

### ❌ “Thread-safe side effects are automatically good.”

No. Avoid unnecessary shared mutable state.

### ❌ “`forEach()` preserves order.”

Not for parallel streams.

### ❌ “Any reduce operation works in parallel.”

The operation needs suitable identity/associative combination semantics.

### ❌ “`groupingByConcurrent()` is always faster.”

No. Benchmark the real workload.

### ❌ “`System.nanoTime()` once is a benchmark.”

Not for reliable JVM microbenchmarking. Use JMH.

---

# 35. Final Revision Sheet 🧠

```text
Parallel Stream
      ↓
Spliterator
      ↓
Partitions
      ↓
Fork/Join tasks
      ↓
Partial results
      ↓
Combine
```

### Use parallel streams when:

```text
Large workload
+ CPU-heavy
+ independent work
+ efficient splitting
+ low synchronization
+ suitable reduction
```

### Avoid / question them when:

```text
Small workload
+ cheap operations
+ blocking I/O
+ shared mutable state
+ strict ordering
+ heavy synchronization
+ common-pool contention
```

### Key APIs

```text
stream()              → sequential
parallelStream()      → parallel
parallel()            → convert stream to parallel
sequential()          → convert back to sequential
forEach()             → no parallel encounter-order guarantee
forEachOrdered()      → preserve encounter order
findFirst()           → first according to encounter order
findAny()             → any matching element
unordered()           → remove ordering requirement when valid
groupingByConcurrent  → concurrent grouping
```

---

# 36. 2-Minute Interview Script 🎤

> “Parallel streams use the stream framework's partitioning mechanism and normally execute through the common ForkJoinPool. They can improve throughput for sufficiently large CPU-bound workloads where elements are independent and the source splits efficiently. However, parallelism introduces scheduling, splitting and combination overhead, so it can be slower for small or cheap workloads. I also avoid shared mutable state and blocking I/O in parallel streams because they can create correctness or common-pool contention problems. For reductions, I ensure the identity and combination operation are suitable for parallel execution, particularly associativity. I use `findAny()` when ordering is irrelevant, while `findFirst()` preserves encounter-order semantics. For production decisions, I benchmark representative workloads rather than assuming parallelism is faster; for JVM microbenchmarks I prefer JMH.”

---

# 🧪 Complete Practice Code

[GitHub — 9.21 Advanced Stream Performance & Parallel Streams Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/21-Advanced-Stream-Performance-and-Parallel-Streams-Deep-Dive)

---

## Navigation

[← 9.20 — `filtering()` / `flatMapping()` / `teeing()`](../20-Collectors-filtering-flatMapping-teeing-Deep-Dive/README.md)

**Current → 9.21 — Advanced Stream Performance & Parallel Streams → ✅ Completed**

**Next → 9.22 — Stream API Best Practices & Anti-Patterns**