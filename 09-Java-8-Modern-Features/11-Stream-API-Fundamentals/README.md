# 9.11 — Java 8 Stream API Fundamentals

> **Interview Goal:** Understand what a Stream is, how a Stream pipeline works, the difference between intermediate and terminal operations, lazy evaluation, one-time consumption, and how Streams differ from Collections.

## 🎯 60-Second Interview Answer

> A Java Stream is a sequence of elements that supports declarative, functional-style processing such as filtering, mapping, sorting, and aggregation. A Stream does not store data; it processes data from a source such as a Collection, array, or generator. Stream operations are divided into intermediate operations, which build a lazy pipeline, and terminal operations, which trigger execution and produce a result or side effect. Streams are single-use and generally do not modify the source collection unless the operations explicitly perform side effects.

---

# 1. What Is a Stream? ⭐⭐⭐⭐⭐

A Stream represents a pipeline for processing a sequence of elements.

Example:

```java
List<Integer> numbers = Arrays.asList(10, 20, 30, 40, 50);

List<Integer> result = numbers.stream()
        .filter(n -> n > 20)
        .collect(Collectors.toList());
```

Conceptually:

```text
Collection
    ↓
 stream()
    ↓
 filter()
    ↓
 collect()
    ↓
 Result
```

A Stream is **not a data structure** that stores elements.

---

# 2. Collection vs Stream ⭐⭐⭐⭐⭐

| Collection | Stream |
|---|---|
| Stores/manages data | Processes data |
| Can be traversed repeatedly | Usually single-use |
| Supports direct mutation depending on type | Does not itself store data |
| External iteration is common | Internal iteration |
| Eager data structure | Lazy pipeline until terminal operation |
| `List`, `Set`, `Map` | `Stream<T>` |

### Interview line

> Collection answers **“How do I store and access data?”**, while Stream answers **“How do I process data?”**

---

# 3. Creating a Stream

## From Collection

```java
List<String> names = Arrays.asList("A", "B", "C");

Stream<String> stream = names.stream();
```

## From Set

```java
Set<Integer> numbers = new HashSet<>();

Stream<Integer> stream = numbers.stream();
```

## From Array

```java
int[] numbers = {10, 20, 30};

IntStream stream = Arrays.stream(numbers);
```

For object arrays:

```java
String[] names = {"Java", "Spring", "Kafka"};

Stream<String> stream = Arrays.stream(names);
```

## From values

```java
Stream<String> stream = Stream.of("Java", "Spring", "Kafka");
```

## Empty Stream

```java
Stream<String> stream = Stream.empty();
```

---

# 4. Stream Pipeline ⭐⭐⭐⭐⭐

A Stream pipeline generally contains:

```text
Source
  ↓
Intermediate Operations
  ↓
Terminal Operation
```

Example:

```java
List<String> result = names.stream()
        .filter(name -> name.length() > 4)
        .map(String::toUpperCase)
        .collect(Collectors.toList());
```

Breakdown:

```text
names.stream()             → Source
filter(...)                → Intermediate
map(...)                   → Intermediate
collect(...)               → Terminal
```

---

# 5. Intermediate Operations ⭐⭐⭐⭐⭐

Intermediate operations return another Stream.

Common examples:

```text
filter()
map()
flatMap()
distinct()
sorted()
limit()
skip()
peek()
```

Example:

```java
Stream<Integer> result = numbers.stream()
        .filter(n -> n > 10)
        .map(n -> n * 2);
```

No terminal operation has been called yet.

Therefore, the pipeline has not necessarily processed the elements.

---

# 6. Terminal Operations ⭐⭐⭐⭐⭐

Terminal operations trigger Stream processing.

Common examples:

```text
collect()
forEach()
count()
reduce()
min()
max()
findFirst()
findAny()
anyMatch()
allMatch()
noneMatch()
```

Example:

```java
long count = numbers.stream()
        .filter(n -> n > 10)
        .count();
```

`count()` triggers execution.

---

# 7. Lazy Evaluation ⭐⭐⭐⭐⭐

This is one of the most important Stream interview concepts.

```java
Stream<Integer> stream = numbers.stream()
        .filter(n -> {
            System.out.println("Filtering: " + n);
            return n > 20;
        });
```

At this point, no filtering output is necessarily produced because there is no terminal operation.

Now:

```java
stream.count();
```

The pipeline executes.

### Mental model

```text
Intermediate operation
        ↓
Build pipeline
        ↓
No terminal operation
        ↓
No pipeline execution
        ↓
Terminal operation
        ↓
Execute pipeline
```

---

# 8. Lazy Evaluation Interview Example 🔥

```java
List<String> names = Arrays.asList("Java", "Spring", "Kafka");

Stream<String> stream = names.stream()
        .filter(name -> {
            System.out.println("filter: " + name);
            return name.length() > 4;
        });

System.out.println("Before terminal operation");

long count = stream.count();

System.out.println("Count: " + count);
```

Expected order:

```text
Before terminal operation
filter: Java
filter: Spring
filter: Kafka
Count: 2
```

The filter runs only when `count()` triggers the pipeline.

---

# 9. Streams Are Single-Use ⭐⭐⭐⭐⭐

A Stream should not be reused after a terminal operation.

```java
Stream<String> stream = Stream.of("Java", "Spring", "Kafka");

stream.count();

stream.forEach(System.out::println);
```

The second operation fails with:

```text
IllegalStateException: stream has already been operated upon or closed
```

If processing is required twice, create a new Stream from the source:

```java
names.stream().count();
names.stream().forEach(System.out::println);
```

### Interview line

> A Stream is consumable and generally single-use; the source can usually be traversed again by creating another Stream.

---

# 10. Stream Does Not Modify the Source by Default

```java
List<Integer> numbers =
        Arrays.asList(10, 20, 30, 40);

List<Integer> result = numbers.stream()
        .filter(n -> n > 20)
        .collect(Collectors.toList());
```

The original list remains:

```text
[10, 20, 30, 40]
```

Result:

```text
[30, 40]
```

Streams describe processing; they do not automatically mutate the source.

---

# 11. `filter()` Fundamentals ⭐⭐⭐⭐⭐

`filter()` keeps elements satisfying a Predicate.

```java
List<Integer> evenNumbers = numbers.stream()
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());
```

Flow:

```text
10 → true  → keep
15 → false → discard
20 → true  → keep
```

Signature conceptually:

```java
Stream<T> filter(Predicate<? super T> predicate)
```

---

# 12. `map()` Fundamentals ⭐⭐⭐⭐⭐

`map()` transforms each element.

```java
List<String> names =
        Arrays.asList("java", "spring", "kafka");

List<String> upperCase = names.stream()
        .map(String::toUpperCase)
        .collect(Collectors.toList());
```

Result:

```text
[JAVA, SPRING, KAFKA]
```

Concept:

```text
T → R
```

---

# 13. `filter()` vs `map()` 🔥

```text
filter()
    ↓
select / remove
    ↓
possibly same number of elements

map()
    ↓
transform
    ↓
usually one output per input
```

Example:

```java
numbers.stream()
        .filter(n -> n > 10)
        .map(n -> n * 2)
        .collect(Collectors.toList());
```

---

# 14. `forEach()`

```java
names.stream()
        .forEach(System.out::println);
```

`forEach()` is terminal.

Avoid using `forEach()` as a replacement for every Stream operation.

Prefer declarative transformations:

```java
List<String> result = names.stream()
        .filter(name -> name.length() > 4)
        .map(String::toUpperCase)
        .collect(Collectors.toList());
```

---

# 15. `sorted()`

Natural ordering:

```java
List<Integer> sorted = numbers.stream()
        .sorted()
        .collect(Collectors.toList());
```

Custom ordering:

```java
List<String> sorted = names.stream()
        .sorted(Comparator.comparing(String::length))
        .collect(Collectors.toList());
```

`sorted()` is intermediate.

---

# 16. `distinct()`

Removes duplicate elements according to equality semantics.

```java
List<Integer> result =
        Arrays.asList(10, 20, 10, 30, 20)
                .stream()
                .distinct()
                .collect(Collectors.toList());
```

Result:

```text
[10, 20, 30]
```

---

# 17. `limit()` and `skip()`

```java
List<Integer> result = numbers.stream()
        .skip(2)
        .limit(3)
        .collect(Collectors.toList());
```

Concept:

```text
skip(2)  → ignore first 2
limit(3) → take next 3
```

Both are intermediate operations.

---

# 18. `count()`

```java
long count = numbers.stream()
        .filter(n -> n > 20)
        .count();
```

Return type:

```java
long
```

not `int`.

---

# 19. `min()` and `max()`

These return Optional because the stream may be empty.

```java
Optional<Integer> min = numbers.stream()
        .min(Integer::compareTo);

Optional<Integer> max = numbers.stream()
        .max(Integer::compareTo);
```

This connects directly to the previous Optional chapter.

---

# 20. `findFirst()` and `findAny()` ⭐⭐⭐⭐⭐

```java
Optional<Integer> first = numbers.stream()
        .filter(n -> n > 20)
        .findFirst();
```

`findFirst()` respects encounter order for ordered streams.

```java
Optional<Integer> any = numbers.stream()
        .filter(n -> n > 20)
        .findAny();
```

`findAny()` is allowed to return any matching element and is especially relevant to parallel processing.

### Interview distinction

```text
findFirst() → first according to encounter order
findAny()   → any matching element
```

---

# 21. `anyMatch()`, `allMatch()`, `noneMatch()`

```java
boolean any = numbers.stream()
        .anyMatch(n -> n > 100);
```

```java
boolean all = numbers.stream()
        .allMatch(n -> n > 0);
```

```java
boolean none = numbers.stream()
        .noneMatch(n -> n < 0);
```

All three are short-circuiting terminal operations.

---

# 22. Short-Circuiting ⭐⭐⭐⭐⭐

Some operations do not necessarily process every element.

Examples:

```text
limit()
findFirst()
findAny()
anyMatch()
allMatch()
noneMatch()
```

Example:

```java
boolean result = Stream.of(1, 2, 3, 4, 5)
        .anyMatch(n -> n > 2);
```

Once `3` matches, the operation can stop.

Mental model:

```text
1 → false
2 → false
3 → true
   ↓
STOP
```

---

# 23. `collect()` Fundamentals ⭐⭐⭐⭐⭐

Common Java 8 collection operation:

```java
List<String> result = names.stream()
        .filter(name -> name.length() > 4)
        .collect(Collectors.toList());
```

Other common collectors:

```text
Collectors.toList()
Collectors.toSet()
Collectors.joining()
Collectors.groupingBy()
Collectors.partitioningBy()
Collectors.counting()
Collectors.summingInt()
```

These will be covered deeply in later topics.

---

# 24. `reduce()` Fundamentals

`reduce()` combines stream elements into one result.

Example:

```java
int sum = numbers.stream()
        .reduce(0, Integer::sum);
```

Flow:

```text
0 + 10 = 10
10 + 20 = 30
30 + 30 = 60
```

Result:

```text
60
```

Another example:

```java
Optional<Integer> sum = numbers.stream()
        .reduce(Integer::sum);
```

Without an identity, an empty Stream produces `Optional.empty()`.

---

# 25. Primitive Streams ⭐⭐⭐⭐⭐

For numeric processing, Java provides:

```text
IntStream
LongStream
DoubleStream
```

Example:

```java
int sum = IntStream.of(10, 20, 30)
        .sum();
```

Average:

```java
OptionalDouble average = IntStream.of(10, 20, 30)
        .average();
```

Primitive streams can avoid unnecessary boxing/unboxing in numeric pipelines.

---

# 26. `mapToInt()`

Given:

```java
List<Employee> employees;
```

Calculate total salary:

```java
int total = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

Concept:

```text
Stream<Employee>
       ↓
mapToInt()
       ↓
IntStream
       ↓
sum()
```

---

# 27. Stream `peek()` — Interview Trap ⚠️

`peek()` is an intermediate operation mainly intended for debugging/inspection.

```java
List<Integer> result = numbers.stream()
        .filter(n -> n > 10)
        .peek(n -> System.out.println("After filter: " + n))
        .map(n -> n * 2)
        .collect(Collectors.toList());
```

Do not rely on `peek()` for essential business side effects.

### Interview line

> `peek()` can be useful for debugging, but business logic should not depend on it because Streams are lazy and execution depends on terminal operations.

---

# 28. Stream Does Not Automatically Mean Faster ⚠️

Do not say:

> “Streams are always faster than loops.”

Correct answer:

> Streams improve declarative readability and can enable parallel processing, but performance depends on the operation, data size, pipeline, source, boxing, and execution mode. A simple loop can be faster for some workloads.

---

# 29. Sequential Stream

```java
numbers.stream()
        .filter(n -> n > 10)
        .forEach(System.out::println);
```

By default, this is sequential.

Conceptually:

```text
one pipeline
one thread for normal sequential execution
```

---

# 30. Parallel Stream Preview

```java
numbers.parallelStream()
        .filter(n -> n > 10)
        .forEach(System.out::println);
```

Parallel streams use the common ForkJoinPool by default.

Important:

```text
parallel ≠ automatically faster
parallel ≠ safe for arbitrary side effects
```

Parallel Streams and their risks will be covered later.

---

# 31. Internal Iteration ⭐⭐⭐⭐⭐

Traditional loop:

```java
for (Integer number : numbers) {
    if (number > 20) {
        System.out.println(number);
    }
}
```

Stream:

```java
numbers.stream()
        .filter(number -> number > 20)
        .forEach(System.out::println);
```

The Stream API controls iteration internally.

### Interview line

> Streams use internal iteration, allowing the library to control traversal and optimize the pipeline, while enhanced for-loops use explicit external iteration logic.

---

# 32. Stream Pipeline Fusion ⭐⭐⭐⭐

Consider:

```java
numbers.stream()
        .filter(n -> n > 10)
        .map(n -> n * 2)
        .filter(n -> n < 100)
        .collect(Collectors.toList());
```

A stream implementation can process an element through the pipeline stages instead of necessarily materializing a separate collection after every operation.

Conceptually:

```text
number
  ↓
filter
  ↓
map
  ↓
filter
  ↓
next number
```

This is one reason lazy pipelines can be efficient.

---

# 33. Complete Practice Program ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.*;

class Employee {

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

public class StreamFundamentalsDemo {

    public static void main(String[] args) {

        List<Integer> numbers = Arrays.asList(
                10, 20, 30, 40, 50, 60);

        List<String> names = Arrays.asList(
                "Java", "Spring", "Kafka", "Docker", "Java");

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000),
                new Employee(2, "Rahul", "HR", 90000),
                new Employee(3, "Priya", "IT", 120000),
                new Employee(4, "Amit", "Finance", 80000));

        // 1. filter()
        List<Integer> greaterThan30 = numbers.stream()
                .filter(n -> n > 30)
                .collect(Collectors.toList());

        // 2. map()
        List<String> upperCase = names.stream()
                .map(String::toUpperCase)
                .collect(Collectors.toList());

        // 3. distinct()
        List<String> uniqueNames = names.stream()
                .distinct()
                .collect(Collectors.toList());

        // 4. sorted()
        List<Integer> sorted = numbers.stream()
                .sorted(Comparator.reverseOrder())
                .collect(Collectors.toList());

        // 5. limit()
        List<Integer> firstThree = numbers.stream()
                .limit(3)
                .collect(Collectors.toList());

        // 6. skip()
        List<Integer> afterTwo = numbers.stream()
                .skip(2)
                .collect(Collectors.toList());

        // 7. count()
        long count = numbers.stream()
                .filter(n -> n > 30)
                .count();

        // 8. findFirst()
        Optional<Integer> first = numbers.stream()
                .filter(n -> n > 25)
                .findFirst();

        // 9. anyMatch()
        boolean hasLargeNumber = numbers.stream()
                .anyMatch(n -> n > 100);

        // 10. allMatch()
        boolean allPositive = numbers.stream()
                .allMatch(n -> n > 0);

        // 11. noneMatch()
        boolean noNegative = numbers.stream()
                .noneMatch(n -> n < 0);

        // 12. reduce()
        int sum = numbers.stream()
                .reduce(0, Integer::sum);

        // 13. Employee filtering + mapping
        List<String> highSalaryEmployees = employees.stream()
                .filter(e -> e.getSalary() > 100000)
                .map(Employee::getName)
                .collect(Collectors.toList());

        // 14. Primitive stream
        int totalSalary = employees.stream()
                .mapToInt(Employee::getSalary)
                .sum();

        // 15. Minimum salary
        Optional<Employee> lowestPaid = employees.stream()
                .min(Comparator.comparingInt(Employee::getSalary));

        // 16. Lazy execution demonstration
        Stream<Integer> lazyStream = numbers.stream()
                .filter(n -> {
                    System.out.println("Filtering: " + n);
                    return n > 40;
                });

        System.out.println("Before terminal operation");

        long lazyCount = lazyStream.count();

        System.out.println("Greater than 30: " + greaterThan30);
        System.out.println("Upper case: " + upperCase);
        System.out.println("Unique names: " + uniqueNames);
        System.out.println("Sorted: " + sorted);
        System.out.println("First three: " + firstThree);
        System.out.println("After two: " + afterTwo);
        System.out.println("Count: " + count);
        System.out.println("First: " + first.orElse(-1));
        System.out.println("Has >100: " + hasLargeNumber);
        System.out.println("All positive: " + allPositive);
        System.out.println("No negative: " + noNegative);
        System.out.println("Sum: " + sum);
        System.out.println("High salary: " + highSalaryEmployees);
        System.out.println("Total salary: " + totalSalary);
        System.out.println("Lowest paid: " + lowestPaid.orElse(null));
        System.out.println("Lazy count: " + lazyCount);
    }
}
```

---

# 34. Interview Output Prediction 🔥

What happens?

```java
Stream.of(1, 2, 3, 4, 5)
        .filter(n -> {
            System.out.println("F" + n);
            return n > 2;
        });

System.out.println("Done");
```

### Answer

Only:

```text
Done
```

The stream has no terminal operation, so the filter pipeline is not triggered.

---

# 35. Output Prediction — Short Circuiting 🔥

```java
boolean result = Stream.of(1, 2, 3, 4, 5)
        .anyMatch(n -> {
            System.out.println(n);
            return n == 3;
        });
```

Expected printed values:

```text
1
2
3
```

Then processing can stop.

Result:

```text
true
```

---

# 36. Output Prediction — Pipeline Order 🔥

```java
Stream.of(1, 2, 3)
        .map(n -> {
            System.out.println("map " + n);
            return n * 2;
        })
        .filter(n -> {
            System.out.println("filter " + n);
            return n > 2;
        })
        .forEach(n -> System.out.println("result " + n));
```

Think element-by-element:

```text
1 → map → filter → discarded
2 → map → filter → result
3 → map → filter → result
```

The exact printed sequence follows that pipeline traversal rather than executing all `map()` calls first and all `filter()` calls afterward.

---

# 37. Common Stream Mistakes ❌

### Mistake 1 — Reusing a Stream

```java
Stream<Integer> stream = numbers.stream();
stream.count();
stream.forEach(System.out::println);
```

❌ Do not reuse after terminal operation.

### Mistake 2 — Expecting intermediate operations to execute immediately

```java
numbers.stream().filter(...);
```

❌ No terminal operation.

### Mistake 3 — Thinking Stream stores data

❌ Stream is a processing abstraction, not a collection.

### Mistake 4 — Assuming Stream automatically mutates the source

❌ It does not by default.

### Mistake 5 — Assuming Streams are always faster

❌ Performance depends on workload.

### Mistake 6 — Using `peek()` for critical business logic

❌ Prefer explicit side-effect operations.

### Mistake 7 — Using `forEach()` when a collector is required

Prefer:

```java
List<String> result = stream.collect(Collectors.toList());
```

when the requirement is to build a collection.

---

# 38. Stream vs Loop — Interview Comparison

| Topic | Loop | Stream |
|---|---|---|
| Style | Imperative | Declarative |
| Iteration | External | Internal |
| Reuse | Loop can run again | Stream normally single-use |
| Lazy processing | No pipeline concept | Yes |
| Readability | Often straightforward | Excellent for transformations |
| Debugging | Often simple | Pipeline can require practice |
| Parallel support | Manual | Built-in API support |
| Performance | Often predictable | Workload-dependent |

### Interview answer

> I choose Streams when the operation is naturally a data-processing pipeline and readability improves. I use loops when control flow is complex, mutation is central, or a loop is clearer and more efficient for the specific workload.

---

# 39. 15 Interview Questions 🔥

### Q1. What is a Stream?

A processing abstraction representing a sequence of elements supporting functional-style operations.

### Q2. Does Stream store data?

No. The source stores data; the Stream processes it.

### Q3. Is Stream reusable?

No, a consumed Stream should not be reused.

### Q4. What are intermediate operations?

Operations returning another Stream and generally evaluated lazily.

### Q5. What are terminal operations?

Operations that trigger pipeline processing and produce a result or side effect.

### Q6. Why are Streams lazy?

To build an execution pipeline and enable optimizations such as short-circuiting and pipeline fusion.

### Q7. `filter()` vs `map()`?

`filter()` selects elements; `map()` transforms elements.

### Q8. What is short-circuiting?

An operation can stop processing once its result is determined.

### Q9. Give short-circuiting examples.

`findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch`, and `limit`.

### Q10. Does a Stream modify the source?

Not by default. Side effects can explicitly modify external state, but that is generally discouraged.

### Q11. `findFirst()` vs `findAny()`?

`findFirst()` respects encounter order for ordered streams; `findAny()` may return any matching element.

### Q12. What does `reduce()` do?

Combines stream elements into a single result.

### Q13. Why use `IntStream`?

For primitive integer processing and to avoid unnecessary boxing in appropriate numeric pipelines.

### Q14. Are Streams always faster than loops?

No.

### Q15. What is the basic Stream pipeline?

Source → intermediate operations → terminal operation.

---

# 40. Coding Challenges 🔥🔥

### Challenge 1
Given:

```java
List<Integer> numbers
```

return all even numbers.

### Challenge 2
Return all numbers greater than 50 and square them.

### Challenge 3
Remove duplicate names and sort them alphabetically.

### Challenge 4
Find the first employee with salary greater than 10 LPA.

### Challenge 5
Check whether any employee belongs to IT.

### Challenge 6
Check whether all employee salaries are positive.

### Challenge 7
Calculate total salary using `mapToInt()`.

### Challenge 8
Find minimum and maximum salary.

### Challenge 9
Calculate average salary.

### Challenge 10 ⭐⭐⭐⭐⭐
Build this pipeline:

```text
Employees
   ↓
filter IT
   ↓
filter salary > 10 LPA
   ↓
map employee name
   ↓
sort
   ↓
collect List<String>
```

### Challenge 11
Demonstrate Stream lazy evaluation using logging inside `filter()`.

### Challenge 12
Demonstrate short-circuiting using `anyMatch()`.

### Challenge 13
Demonstrate why a Stream cannot be reused.

### Challenge 14
Convert `List<Integer>` to `IntStream` and calculate sum, average, min, and max.

### Challenge 15 — Interview Level ⭐⭐⭐⭐⭐
Explain every stage of a complex Stream pipeline in under 2 minutes and identify:

```text
source
intermediate operations
terminal operation
lazy operations
short-circuiting operation
final result
```

---

# 41. Final Revision Sheet 🧠

```text
STREAM
────────────────────────────────────
Stream = data processing abstraction
Stream does NOT store data
Usually single-use
Supports functional-style processing
```

```text
PIPELINE
────────────────────────────────────
Source
  ↓
Intermediate operations
  ↓
Terminal operation
```

```text
INTERMEDIATE
────────────────────────────────────
filter()
map()
flatMap()
distinct()
sorted()
skip()
limit()
peek()
```

```text
TERMINAL
────────────────────────────────────
collect()
forEach()
count()
reduce()
min()
max()
findFirst()
findAny()
anyMatch()
allMatch()
noneMatch()
```

```text
IMPORTANT
────────────────────────────────────
filter() → select
map() → transform
reduce() → combine
count() → count
findFirst() → first
findAny() → any
```

```text
REMEMBER
────────────────────────────────────
Lazy until terminal operation
Single-use
Short-circuiting exists
Not automatically faster
Doesn't automatically mutate source
```

---

# 42. 2-Minute Interview Script 🎤

> “Java Stream API, introduced in Java 8, provides a declarative way to process data from sources such as Collections, arrays, and generators. A Stream doesn't store data; it builds a processing pipeline. The pipeline consists of a source, zero or more intermediate operations, and a terminal operation. Intermediate operations such as filter, map, sorted, and distinct are lazy and return another Stream. The terminal operation such as collect, count, reduce, or forEach triggers execution. Streams are generally single-use. They also support short-circuiting operations such as anyMatch and findFirst, which can stop processing early. I don't assume Streams are always faster than loops; I choose them mainly when they make data-processing logic clearer, while considering performance and side effects. For numeric processing, primitive streams such as IntStream can reduce unnecessary boxing.”

---

# 🧪 Complete Practice Code

urlGitHub — 9.11 Java 8 Stream API Fundamentals Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/11-Stream-API-Fundamentals

---

## Navigation

[← 9.10 — Optional Interview Scenarios & Final Revision](../10-Optional-Interview-Scenarios-Final-Revision/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.11 — Java 8 Stream API Fundamentals → ✅ Completed**

**Next → 9.12 — Stream `filter()` / `map()` / `flatMap()` Deep Dive**