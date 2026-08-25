# 9.14 — Stream `reduce()` / `collect()` Fundamentals

## 🎯 Interview Goal

Understand the two operations that are frequently confused in Java Stream interviews:

```text
reduce()  → combine stream elements into one result
collect() → accumulate stream elements into a mutable/container result
```

---

# 1. `reduce()` Fundamentals ⭐⭐⭐⭐⭐

`reduce()` combines stream elements into a single result.

Simple sum:

```java
int sum = Arrays.asList(10, 20, 30, 40)
        .stream()
        .reduce(0, (a, b) -> a + b);
```

Output:

```text
100
```

Mental model:

```text
10 + 20 + 30 + 40
        ↓
      100
```

---

# 2. `reduce()` With Method Reference

```java
int sum = numbers.stream()
        .reduce(0, Integer::sum);
```

The accumulator combines two values:

```text
current result + next element
```

---

# 3. `reduce()` Without Identity ⭐⭐⭐⭐⭐

There is an overload that returns `Optional<T>`:

```java
Optional<Integer> sum = numbers.stream()
        .reduce(Integer::sum);
```

Why `Optional`?

Because the stream may be empty and there may be no result.

```java
Optional<Integer> result = Stream.<Integer>empty()
        .reduce(Integer::sum);
```

---

# 4. Identity Value

Example:

```java
int sum = numbers.stream()
        .reduce(0, Integer::sum);
```

Here:

```text
0 → identity
Integer::sum → accumulator
```

For addition:

```text
identity = 0
```

For multiplication:

```text
identity = 1
```

Example:

```java
int product = Arrays.asList(2, 3, 4)
        .stream()
        .reduce(1, (a, b) -> a * b);
```

Result:

```text
24
```

---

# 5. `reduce()` to Find Maximum

```java
Optional<Integer> max = numbers.stream()
        .reduce(Integer::max);
```

Or:

```java
int max = numbers.stream()
        .reduce(Integer.MIN_VALUE, Integer::max);
```

---

# 6. `reduce()` to Find Minimum

```java
Optional<Integer> min = numbers.stream()
        .reduce(Integer::min);
```

---

# 7. String Concatenation

```java
String result = Arrays.asList("Java", "Spring", "Kafka")
        .stream()
        .reduce("", (a, b) -> a + b);
```

Output:

```text
JavaSpringKafka
```

With separator:

```java
String result = Arrays.asList("Java", "Spring", "Kafka")
        .stream()
        .reduce("", (a, b) -> a.isEmpty() ? b : a + ", " + b);
```

---

# 8. `reduce()` Employee Example ⭐⭐⭐⭐⭐

Find total salary:

```java
int totalSalary = employees.stream()
        .map(Employee::getSalary)
        .reduce(0, Integer::sum);
```

Pipeline:

```text
Employee
   ↓
map salary
   ↓
Integer Stream
   ↓
reduce sum
   ↓
Total salary
```

---

# 9. `reduce()` vs `collect()` 🔥🔥🔥

This is one of the most important interview questions.

### `reduce()`

Used when the goal is one combined value:

```text
numbers → sum
numbers → max
numbers → min
values → product
```

### `collect()`

Used when the goal is to accumulate into a result container:

```text
Stream → List
Stream → Set
Stream → Map
Stream → String
```

### 2-second interview answer

> `reduce()` combines stream elements into a single value, while `collect()` performs mutable reduction into a result container such as a List, Set, Map, or custom collector.

---

# 10. `collect(Collectors.toList())` ⭐⭐⭐⭐⭐

```java
List<Integer> result = numbers.stream()
        .filter(n -> n > 10)
        .collect(Collectors.toList());
```

The stream elements are accumulated into a List.

---

# 11. `collect(Collectors.toSet())`

Remove duplicates while collecting:

```java
Set<Integer> result = numbers.stream()
        .collect(Collectors.toSet());
```

Important: a normal `Set` does not guarantee sorted order.

If sorted unique values are required:

```java
Set<Integer> result = numbers.stream()
        .collect(Collectors.toCollection(TreeSet::new));
```

---

# 12. `collect()` to Map ⭐⭐⭐⭐⭐

Employee ID → Employee:

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                employee -> employee
        ));
```

Shorter value mapper:

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity()
        ));
```

---

# 13. Duplicate Keys — Important Trap ⚠️

This can fail:

```java
employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity()
        ));
```

If multiple employees have the same department, duplicate keys cause an `IllegalStateException`.

Use a merge function:

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                (e1, e2) -> e1
        ));
```

### Interview point ⭐⭐⭐⭐⭐

> `Collectors.toMap()` requires unique keys unless a merge function is supplied.

---

# 14. Grouping With `groupingBy()` 🔥🔥🔥

Group employees by department:

```java
Map<String, List<Employee>> grouped = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Result conceptually:

```text
IT       → [Employee1, Employee2]
HR       → [Employee3]
Finance  → [Employee4]
```

---

# 15. Grouping + Counting

Count employees per department:

```java
Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

---

# 16. Grouping + Mapping ⭐⭐⭐⭐⭐

Department → employee names:

```java
Map<String, List<String>> namesByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.toList()
                )
        ));
```

Flow:

```text
Employee
 ↓
group by department
 ↓
map employee → name
 ↓
collect List<String>
```

---

# 17. Partitioning With `partitioningBy()`

Split into two groups based on a boolean condition:

```java
Map<Boolean, List<Employee>> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() > 100000
        ));
```

Result:

```text
true  → salary > 100000
false → salary <= 100000
```

### `groupingBy()` vs `partitioningBy()`

```text
groupingBy()      → many possible keys
partitioningBy()  → true / false partitions
```

---

# 18. `joining()`

```java
String names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining(", "));
```

Output:

```text
Nirbhay, Rahul, Priya
```

With prefix and suffix:

```java
String result = names.stream()
        .collect(Collectors.joining(", ", "[", "]"));
```

---

# 19. Summarizing Collectors ⭐⭐⭐⭐⭐

```java
IntSummaryStatistics stats = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary));
```

Now you can access:

```java
stats.getCount();
stats.getSum();
stats.getMin();
stats.getMax();
stats.getAverage();
```

This is often cleaner than performing multiple independent terminal operations.

---

# 20. `summingInt()`

```java
int totalSalary = employees.stream()
        .collect(Collectors.summingInt(Employee::getSalary));
```

Compare with:

```java
int totalSalary = employees.stream()
        .map(Employee::getSalary)
        .reduce(0, Integer::sum);
```

Both can calculate a sum, but the collector expresses aggregation intent directly.

---

# 21. `averagingInt()`

```java
double averageSalary = employees.stream()
        .collect(Collectors.averagingInt(Employee::getSalary));
```

---

# 22. `maxBy()` / `minBy()`

Highest-paid employee:

```java
Optional<Employee> highestPaid = employees.stream()
        .collect(Collectors.maxBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

Similarly:

```java
Optional<Employee> lowestPaid = employees.stream()
        .collect(Collectors.minBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

---

# 23. `reduce()` vs `collect()` Example 🔥

### Total salary with reduce

```java
int total = employees.stream()
        .map(Employee::getSalary)
        .reduce(0, Integer::sum);
```

### Total salary with collector

```java
int total = employees.stream()
        .collect(Collectors.summingInt(Employee::getSalary));
```

### Interview explanation

> Both can aggregate values, but `reduce()` is the general associative reduction mechanism for combining values, whereas `collect()` is designed for mutable result accumulation and has dedicated collectors for common container and aggregation operations.

---

# 24. Why Not Use `reduce()` to Build a List? ⚠️

You technically can write reduction logic that builds a list, but it is generally the wrong abstraction:

```java
List<Integer> result = numbers.stream()
        .reduce(
                new ArrayList<>(),
                (list, n) -> {
                    list.add(n);
                    return list;
                },
                (list1, list2) -> {
                    list1.addAll(list2);
                    return list1;
                }
        );
```

Prefer:

```java
List<Integer> result = numbers.stream()
        .collect(Collectors.toList());
```

### Interview point

> `reduce()` is intended for value reduction; `collect()` is the natural abstraction for accumulating into mutable containers.

---

# 25. Three-Argument `reduce()` ⭐⭐⭐⭐⭐

The overload:

```java
<U> U reduce(
    U identity,
    BiFunction<U, ? super T, U> accumulator,
    BinaryOperator<U> combiner
)
```

Example:

```java
int result = numbers.parallelStream()
        .reduce(
                0,
                (subtotal, value) -> subtotal + value,
                Integer::sum
        );
```

Roles:

```text
identity    → starting value
accumulator → combine current partial result with an element
combiner    → combine partial results
```

---

# 26. Why `combiner` Matters 🔥

For a sequential stream, the combiner may not be visibly involved in the same way as with parallel reduction.

For a parallel stream:

```text
Partition 1 → partial result
Partition 2 → partial result
Partition 3 → partial result
        ↓
    combiner
        ↓
 final result
```

The reduction functions must obey the required associativity/identity rules for correct parallel behavior.

---

# 27. `reduce()` Must Be Suitable for Parallel Use ⚠️

Avoid reductions whose accumulator depends on mutable shared state.

Bad design:

```java
List<Integer> shared = new ArrayList<>();

numbers.parallelStream()
        .forEach(shared::add);
```

This is not a safe general pattern.

Prefer collectors designed for parallel accumulation or thread-safe designs where appropriate.

---

# 28. `collect()` and Parallel Streams

A collector can provide separate accumulation containers for parallel processing and then combine them.

Conceptually:

```text
Thread 1 → container A
Thread 2 → container B
Thread 3 → container C
             ↓
          combiner
             ↓
        final result
```

This is one reason `collect()` is preferred for mutable accumulation.

---

# 29. `Collector` Mental Model

A Collector can be understood through these major pieces:

```text
supplier   → creates result container
accumulator → adds one element
combiner   → merges partial containers
finisher   → converts final intermediate result
characteristics → describes collector behavior
```

Example:

```java
Collectors.toList()
```

hides these mechanics from application code.

---

# 30. Custom Collector — Conceptual Example

```java
Collector<String, StringBuilder, String> collector =
        Collector.of(
                StringBuilder::new,
                (builder, value) -> {
                    if (builder.length() > 0) {
                        builder.append(", ");
                    }
                    builder.append(value);
                },
                (left, right) -> {
                    if (left.length() == 0) return right;
                    if (right.length() == 0) return left;
                    return left.append(", ").append(right);
                },
                StringBuilder::toString
        );

String result = Arrays.asList("Java", "Spring", "Kafka")
        .stream()
        .collect(collector);
```

Result:

```text
Java, Spring, Kafka
```

This is useful for understanding what `collect()` does internally at a conceptual level.

---

# 31. `toList()` vs `toCollection()`

Use `toCollection()` when you specifically need a particular collection implementation:

```java
LinkedList<Integer> result = numbers.stream()
        .collect(Collectors.toCollection(LinkedList::new));
```

Sorted set:

```java
TreeSet<Integer> result = numbers.stream()
        .collect(Collectors.toCollection(TreeSet::new));
```

---

# 32. Complete Real-World Pipeline ⭐⭐⭐⭐⭐

Requirement:

> Group high-paid IT employees by department and return their names sorted by salary descending.

```java
Map<String, List<String>> result = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                LinkedHashMap::new,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.toList()
                )
        ));
```

This combines filtering, sorting, grouping, mapping and collection.

---

# 33. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.IntStream;
import java.util.stream.Stream;

public class ReduceCollectDemo {

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

        List<Integer> numbers = Arrays.asList(10, 20, 30, 40, 20);

        // 1. Sum using reduce
        int sum = numbers.stream()
                .reduce(0, Integer::sum);

        // 2. Product using reduce
        int product = Arrays.asList(2, 3, 4).stream()
                .reduce(1, (a, b) -> a * b);

        // 3. Max using reduce
        Optional<Integer> max = numbers.stream()
                .reduce(Integer::max);

        // 4. Min using reduce
        Optional<Integer> min = numbers.stream()
                .reduce(Integer::min);

        // 5. Reduce without identity
        Optional<Integer> optionalSum = numbers.stream()
                .reduce(Integer::sum);

        // 6. Collect to List
        List<Integer> list = numbers.stream()
                .filter(n -> n > 15)
                .collect(Collectors.toList());

        // 7. Collect to Set
        Set<Integer> set = numbers.stream()
                .collect(Collectors.toSet());

        // 8. Collect to TreeSet
        TreeSet<Integer> sortedSet = numbers.stream()
                .collect(Collectors.toCollection(TreeSet::new));

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000),
                new Employee(2, "Rahul", "IT", 120000),
                new Employee(3, "Priya", "HR", 130000),
                new Employee(4, "Amit", "IT", 150000),
                new Employee(5, "Sneha", "Finance", 110000),
                new Employee(6, "Ravi", "HR", 90000)
        );

        // 9. Total salary using reduce
        int totalSalary = employees.stream()
                .map(Employee::getSalary)
                .reduce(0, Integer::sum);

        // 10. Total salary using collector
        int totalSalaryUsingCollector = employees.stream()
                .collect(Collectors.summingInt(Employee::getSalary));

        // 11. Employee ID -> Employee
        Map<Integer, Employee> byId = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getId,
                        Function.identity()
                ));

        // 12. Group by department
        Map<String, List<Employee>> byDepartment = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment));

        // 13. Count by department
        Map<String, Long> countByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.counting()
                ));

        // 14. Department -> employee names
        Map<String, List<String>> namesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        )
                ));

        // 15. Partition by salary
        Map<Boolean, List<Employee>> salaryPartition = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() > 100000
                ));

        // 16. Join employee names
        String names = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.joining(", "));

        // 17. Salary statistics
        IntSummaryStatistics stats = employees.stream()
                .collect(Collectors.summarizingInt(Employee::getSalary));

        // 18. Average salary
        double averageSalary = employees.stream()
                .collect(Collectors.averagingInt(Employee::getSalary));

        // 19. Highest-paid employee
        Optional<Employee> highestPaid = employees.stream()
                .collect(Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                ));

        // 20. High-paid IT employees grouped by department
        Map<String, List<String>> highPaidIT = employees.stream()
                .filter(e -> "IT".equals(e.getDepartment()))
                .filter(e -> e.getSalary() > 100000)
                .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        LinkedHashMap::new,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        )
                ));

        // 21. Three-argument reduce
        int parallelSum = numbers.parallelStream()
                .reduce(
                        0,
                        Integer::sum,
                        Integer::sum
                );

        System.out.println("Sum: " + sum);
        System.out.println("Product: " + product);
        System.out.println("Max: " + max);
        System.out.println("Min: " + min);
        System.out.println("Optional sum: " + optionalSum);
        System.out.println("List: " + list);
        System.out.println("Set: " + set);
        System.out.println("Sorted set: " + sortedSet);
        System.out.println("Total salary: " + totalSalary);
        System.out.println("Total salary collector: " + totalSalaryUsingCollector);
        System.out.println("By ID: " + byId);
        System.out.println("By department: " + byDepartment);
        System.out.println("Count by department: " + countByDepartment);
        System.out.println("Names by department: " + namesByDepartment);
        System.out.println("Salary partition: " + salaryPartition);
        System.out.println("Names: " + names);
        System.out.println("Count: " + stats.getCount());
        System.out.println("Sum: " + stats.getSum());
        System.out.println("Min salary: " + stats.getMin());
        System.out.println("Max salary: " + stats.getMax());
        System.out.println("Average salary: " + stats.getAverage());
        System.out.println("Average salary direct: " + averageSalary);
        System.out.println("Highest paid: " + highestPaid);
        System.out.println("High-paid IT: " + highPaidIT);
        System.out.println("Parallel sum: " + parallelSum);
    }
}
```

---

# 34. Output Prediction 🔥

What is the output?

```java
int result = Arrays.asList(1, 2, 3, 4, 5)
        .stream()
        .reduce(0, (a, b) -> a + b);
```

Answer:

```text
15
```

---

# 35. Output Prediction — Empty Stream ⭐⭐⭐⭐⭐

```java
Optional<Integer> result = Stream.<Integer>empty()
        .reduce(Integer::sum);
```

Answer:

```text
Optional.empty()
```

Because there is no identity and no element to produce a result.

---

# 36. `toMap()` Interview Trap 🔥

What happens?

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity()
        ));
```

If multiple employees belong to the same department:

```text
IllegalStateException
```

Correct with merge function:

```java
.collect(Collectors.toMap(
        Employee::getDepartment,
        Function.identity(),
        (e1, e2) -> e1
));
```

---

# 37. `groupingBy()` vs `toMap()` 🔥🔥🔥

### Need one value per key

```java
Collectors.toMap(keyMapper, valueMapper)
```

### Need multiple values per key

```java
Collectors.groupingBy(keyMapper)
```

Example:

```text
Department → Employee
```
may require `toMap()` only if department is unique.

```text
Department → List<Employee>
```
should use `groupingBy()`.

---

# 38. 20 Interview Questions 🔥🔥🔥

1. What is `reduce()`?
2. What is the identity in `reduce()`?
3. Why does `reduce()` without identity return `Optional`?
4. What is the accumulator?
5. What is the combiner in three-argument `reduce()`?
6. Why is the combiner important for parallel streams?
7. What is the difference between `reduce()` and `collect()`?
8. Why is `collect()` preferred for mutable containers?
9. How do you calculate sum using `reduce()`?
10. How do you find max using `reduce()`?
11. How do you collect a Stream into a List?
12. How do you collect into a Set?
13. How do you collect into a Map?
14. What happens if `toMap()` gets duplicate keys?
15. How does the merge function solve duplicate keys?
16. Difference between `groupingBy()` and `partitioningBy()`?
17. How do you group employees by department?
18. How do you calculate average salary using collectors?
19. What is `IntSummaryStatistics`?
20. Explain `reduce()` vs `collect()` in 2 minutes.

---

# 39. Coding Challenges 🔥🔥🔥

### Challenge 1
Calculate sum using `reduce()`.

### Challenge 2
Calculate product using `reduce()`.

### Challenge 3
Find maximum and minimum using `reduce()`.

### Challenge 4
Calculate total employee salary.

### Challenge 5
Collect even numbers into a List.

### Challenge 6
Collect unique numbers into a Set.

### Challenge 7
Convert employees into `Map<Integer, Employee>`.

### Challenge 8
Group employees by department.

### Challenge 9
Count employees per department.

### Challenge 10
Department → employee names.

### Challenge 11
Partition employees by salary > 10 LPA.

### Challenge 12
Create a comma-separated employee-name String.

### Challenge 13
Find average salary using a collector.

### Challenge 14
Find highest-paid employee using `maxBy()`.

### Challenge 15 ⭐⭐⭐⭐⭐
Group employees by department and calculate total salary per department.

### Challenge 16 ⭐⭐⭐⭐⭐
Find the highest salary in each department.

### Challenge 17 ⭐⭐⭐⭐⭐
Return department → sorted employee names by salary descending.

### Challenge 18
Solve a duplicate-key `toMap()` problem using a merge function.

### Challenge 19
Implement a three-argument parallel `reduce()` correctly.

### Challenge 20 — 5-Year Interview Level 🔥
Explain when you would choose `reduce()`, `collect()`, `groupingBy()`, `toMap()`, and `summarizingInt()` in a production Java service.

---

# 40. Common Mistakes ❌

### ❌ Mistake 1
Using `reduce()` when you really need a List/Set/Map.

Prefer `collect()` for mutable result containers.

### ❌ Mistake 2
Using a wrong identity.

For sum:

```text
0
```

For multiplication:

```text
1
```

### ❌ Mistake 3
Ignoring duplicate keys in `toMap()`.

### ❌ Mistake 4
Using shared mutable state with parallel streams.

### ❌ Mistake 5
Assuming `groupingBy()` and `toMap()` solve exactly the same problem.

### ❌ Mistake 6
Forgetting that a reduction intended for parallel execution must satisfy the required identity/associativity rules.

---

# 41. Final Revision Sheet 🧠

```text
reduce()
────────────────────────
Many elements → one value
Examples:
sum / product / min / max
```

```text
collect()
────────────────────────
Stream → result container/aggregation
Examples:
List / Set / Map / grouping / joining
```

```text
toMap()
────────────────────────
key → value
Duplicate keys require merge handling
```

```text
groupingBy()
────────────────────────
key → List/value aggregation
```

```text
partitioningBy()
────────────────────────
true / false groups
```

### Golden Rule

```text
Need ONE combined value?       → reduce()
Need a container?              → collect()
Need key → value?              → toMap()
Need key → multiple values?    → groupingBy()
Need true/false groups?        → partitioningBy()
Need sum/avg/statistics?       → specialized collectors
```

---

# 42. 2-Minute Interview Script 🎤

> “`reduce()` and `collect()` are both terminal operations, but they solve different aggregation problems. `reduce()` combines stream elements into a single result, such as a sum, product, minimum or maximum. It can have an identity and accumulator, and the three-argument form also has a combiner for parallel reduction. `collect()` is designed for mutable result accumulation and is commonly used to create Lists, Sets, Maps, grouped results and strings. For example, I use `reduce()` for total salary when I want a single value, while I use `groupingBy()` when I need department-to-employees. With `toMap()`, duplicate keys must be handled with a merge function. For production code, I choose the operation based on the shape of the required result rather than forcing every aggregation into `reduce()`.”

---

# 🧪 Complete Practice Code

[GitHub — 9.14 Stream reduce/collect Fundamentals Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/14-Stream-reduce-collect-Fundamentals)

---

## Navigation

[← 9.13 — Stream sorted/distinct/limit/skip](../13-Stream-sorted-distinct-limit-skip/README.md)

**Current → 9.14 — Stream `reduce()` / `collect()` Fundamentals → ✅ Completed**

**Next → 9.15 — `Collectors.groupingBy()` / `partitioningBy()` Deep Dive**