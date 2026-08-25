# 9.13 — Stream `sorted()` / `distinct()` / `limit()` / `skip()`

## 🎯 Interview Goal

Master four common Stream operations used for ordering, duplicate removal and pagination-style processing.

```text
sorted()   → order elements
 distinct() → remove duplicates
 limit(n)   → take first n elements
 skip(n)    → ignore first n elements
```

---

# 1. `sorted()` Fundamentals ⭐⭐⭐⭐⭐

Sort elements in natural order:

```java
List<Integer> result = Arrays.asList(40, 10, 30, 20)
        .stream()
        .sorted()
        .collect(Collectors.toList());
```

Output:

```text
[10, 20, 30, 40]
```

`sorted()` without an argument uses the elements' natural ordering.

Conceptually:

```java
Stream<T> → Stream<T>
```

---

# 2. `sorted(Comparator)` ⭐⭐⭐⭐⭐

Descending numbers:

```java
List<Integer> result = numbers.stream()
        .sorted(Comparator.reverseOrder())
        .collect(Collectors.toList());
```

Employee salary descending:

```java
List<Employee> result = employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .collect(Collectors.toList());
```

### Interview answer

> `sorted()` uses natural ordering, while `sorted(Comparator)` allows custom ordering.

---

# 3. Sorting Strings

```java
List<String> result = Arrays.asList("Spring", "Java", "Kafka")
        .stream()
        .sorted()
        .collect(Collectors.toList());
```

For case-insensitive ordering:

```java
.sorted(String.CASE_INSENSITIVE_ORDER)
```

---

# 4. Sorting Objects 🔥

```java
List<Employee> result = employees.stream()
        .sorted(Comparator.comparing(Employee::getName))
        .collect(Collectors.toList());
```

By salary:

```java
.sorted(Comparator.comparingInt(Employee::getSalary))
```

By salary descending:

```java
.sorted(Comparator.comparingInt(Employee::getSalary).reversed())
```

---

# 5. Multiple Sort Criteria ⭐⭐⭐⭐⭐

Requirement:

> Sort employees by department, then salary descending, then name.

```java
List<Employee> result = employees.stream()
        .sorted(Comparator.comparing(Employee::getDepartment)
                .thenComparing(Comparator.comparingInt(Employee::getSalary).reversed())
                .thenComparing(Employee::getName))
        .collect(Collectors.toList());
```

Mental model:

```text
department
    ↓ tie
salary DESC
    ↓ tie
name ASC
```

---

# 6. `distinct()` Fundamentals ⭐⭐⭐⭐⭐

Remove duplicate values:

```java
List<Integer> result = Arrays.asList(10, 20, 10, 30, 20)
        .stream()
        .distinct()
        .collect(Collectors.toList());
```

Output:

```text
[10, 20, 30]
```

`distinct()` uses equality semantics (`equals()` / `hashCode()`) to determine duplicates.

---

# 7. `distinct()` With Objects — Important Trap ⚠️

For custom objects, `distinct()` depends on correctly implemented `equals()` and `hashCode()`.

```java
class Employee {
    private int id;
    private String name;

    // equals() and hashCode() should be based on the chosen identity fields.
}
```

If two Employee objects have the same id but `equals()` still uses object identity, `distinct()` may keep both.

### Interview answer

> `distinct()` is stateful and uses equality semantics. For custom objects, correct `equals()` and `hashCode()` implementations are essential.

---

# 8. `limit()` Fundamentals ⭐⭐⭐⭐⭐

Take only the first N elements:

```java
List<Integer> result = numbers.stream()
        .limit(3)
        .collect(Collectors.toList());
```

If the stream is:

```text
10, 20, 30, 40, 50
```

then:

```text
limit(3)
↓
10, 20, 30
```

Conceptually:

```text
N = 3
Keep first 3
```

---

# 9. `skip()` Fundamentals ⭐⭐⭐⭐⭐

Ignore the first N elements:

```java
List<Integer> result = numbers.stream()
        .skip(2)
        .collect(Collectors.toList());
```

Input:

```text
10, 20, 30, 40, 50
```

Output:

```text
[30, 40, 50]
```

---

# 10. `skip()` + `limit()` = Pagination 🔥🔥🔥

For page number and page size:

```java
int page = 3;
int pageSize = 10;

List<Employee> result = employees.stream()
        .skip((long) (page - 1) * pageSize)
        .limit(pageSize)
        .collect(Collectors.toList());
```

Formula:

```text
offset = (page - 1) × pageSize
```

For page 3, size 10:

```text
skip(20)
limit(10)
```

### Interview nuance

For large database-backed datasets, database pagination (`OFFSET/LIMIT`, keyset pagination, etc.) is generally preferable to loading the entire dataset and then using Stream `skip()`.

---

# 11. Operation Order Matters 🔥🔥🔥

These are different:

```java
numbers.stream()
        .distinct()
        .limit(3)
```

vs

```java
numbers.stream()
        .limit(3)
        .distinct()
```

Example:

```text
Input: 1, 1, 2, 2, 3, 3
```

First:

```text
distinct → 1,2,3
limit(3) → 1,2,3
```

Second:

```text
limit(3) → 1,1,2
distinct → 1,2
```

So **pipeline order can change the result**.

---

# 12. `sorted()` + `limit()` — Top N 🔥

Top 3 highest salaries:

```java
List<Employee> top3 = employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .limit(3)
        .collect(Collectors.toList());
```

Pipeline:

```text
Employees
 ↓
sort salary DESC
 ↓
limit 3
 ↓
Top 3
```

---

# 13. `sorted()` + `skip()` + `limit()`

Get employees ranked 6–10 by salary:

```java
List<Employee> result = employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .skip(5)
        .limit(5)
        .collect(Collectors.toList());
```

This is a classic interview exercise.

---

# 14. `distinct()` + `sorted()`

Unique salaries in ascending order:

```java
List<Integer> salaries = employees.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
```

Or sort first:

```java
.map(Employee::getSalary)
.sorted()
.distinct()
```

Both produce the same ordered unique result, but their internal work can differ. In an interview, explain the trade-off rather than claiming one order is universally faster.

---

# 15. Top N Distinct Values ⭐⭐⭐⭐⭐

Top 3 distinct salaries:

```java
List<Integer> top3Salaries = employees.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .limit(3)
        .collect(Collectors.toList());
```

This is different from top 3 employees because duplicate salaries are removed first.

---

# 16. `filter()` + `sorted()` + `limit()`

Top 5 IT employees earning more than 10 LPA:

```java
List<Employee> result = employees.stream()
        .filter(e -> "IT".equals(e.getDepartment()))
        .filter(e -> e.getSalary() > 100000)
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .limit(5)
        .collect(Collectors.toList());
```

This is a realistic interview pipeline.

---

# 17. `skip()` + `limit()` With Sorted Data

Second-highest to fifth-highest salary:

```java
List<Employee> result = employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .skip(1)
        .limit(4)
        .collect(Collectors.toList());
```

Remember:

```text
skip(1)  → ignore rank 1
limit(4) → take next 4
```

---

# 18. Short-Circuiting ⭐⭐⭐⭐⭐

`limit()` and `skip()` are short-circuit/stateful operations with important effects on how much of a pipeline must be processed.

Example:

```java
Stream.iterate(1, n -> n + 1)
        .limit(5)
        .forEach(System.out::println);
```

Output:

```text
1
2
3
4
5
```

Without a limiting operation, the source is infinite.

---

# 19. `limit()` With Infinite Stream

```java
List<Integer> result = Stream.iterate(1, n -> n + 1)
        .limit(10)
        .collect(Collectors.toList());
```

Result:

```text
[1,2,3,4,5,6,7,8,9,10]
```

This is one reason `limit()` is especially important for generated streams.

---

# 20. `skip()` With Infinite Stream

```java
List<Integer> result = Stream.iterate(1, n -> n + 1)
        .skip(5)
        .limit(5)
        .collect(Collectors.toList());
```

Result:

```text
[6,7,8,9,10]
```

---

# 21. `sorted()` Is Stateful ⚠️

To determine the first element of a sorted stream, the implementation generally needs to consider the relevant source elements before producing the sorted result.

Therefore, don't think of `sorted()` as a simple one-element-at-a-time operation like `map()`.

### Interview line

> `sorted()` is a stateful intermediate operation because it needs ordering information from the stream elements.

---

# 22. `distinct()` Is Stateful ⚠️

`distinct()` needs to remember previously encountered values to determine whether a later element is a duplicate.

```text
10 → keep
20 → keep
10 → discard
30 → keep
20 → discard
```

### Interview line

> `distinct()` is stateful because it must maintain information about elements already seen.

---

# 23. `limit()` vs `skip()`

| Operation | Meaning |
|---|---|
| `limit(n)` | Keep first n elements |
| `skip(n)` | Discard first n elements |
| `skip(n).limit(m)` | Pagination/window |

---

# 24. `sorted()` vs `Comparator`

Natural ordering:

```java
.sorted()
```

Custom ordering:

```java
.sorted(Comparator.comparing(Employee::getName))
```

Reverse ordering:

```java
.sorted(Comparator.reverseOrder())
```

Object reverse ordering:

```java
.sorted(Comparator.comparing(Employee::getSalary).reversed())
```

---

# 25. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class SortedDistinctLimitSkipDemo {

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

        List<Integer> numbers = Arrays.asList(40, 10, 30, 20, 10, 50, 30);

        // 1. Natural sorting
        List<Integer> ascending = numbers.stream()
                .sorted()
                .collect(Collectors.toList());

        // 2. Reverse sorting
        List<Integer> descending = numbers.stream()
                .sorted(Comparator.reverseOrder())
                .collect(Collectors.toList());

        // 3. Distinct values
        List<Integer> unique = numbers.stream()
                .distinct()
                .collect(Collectors.toList());

        // 4. First 3 elements
        List<Integer> firstThree = numbers.stream()
                .limit(3)
                .collect(Collectors.toList());

        // 5. Skip first 3
        List<Integer> afterThree = numbers.stream()
                .skip(3)
                .collect(Collectors.toList());

        // 6. Top 3 distinct numbers
        List<Integer> topThreeDistinct = numbers.stream()
                .distinct()
                .sorted(Comparator.reverseOrder())
                .limit(3)
                .collect(Collectors.toList());

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000),
                new Employee(2, "Rahul", "IT", 120000),
                new Employee(3, "Priya", "HR", 130000),
                new Employee(4, "Amit", "IT", 150000),
                new Employee(5, "Sneha", "IT", 110000),
                new Employee(6, "Ravi", "Finance", 100000),
                new Employee(7, "Ankit", "IT", 90000)
        );

        // 7. Employees sorted by salary descending
        List<Employee> salaryDesc = employees.stream()
                .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
                .collect(Collectors.toList());

        // 8. Top 3 employees by salary
        List<Employee> topThreeEmployees = employees.stream()
                .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
                .limit(3)
                .collect(Collectors.toList());

        // 9. Top 3 distinct salaries
        List<Integer> topThreeSalaries = employees.stream()
                .map(Employee::getSalary)
                .distinct()
                .sorted(Comparator.reverseOrder())
                .limit(3)
                .collect(Collectors.toList());

        // 10. Employees ranked 4 to 6 by salary
        List<Employee> rankFourToSix = employees.stream()
                .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
                .skip(3)
                .limit(3)
                .collect(Collectors.toList());

        // 11. Top 5 IT employees earning > 10 LPA
        List<Employee> topFiveIT = employees.stream()
                .filter(e -> "IT".equals(e.getDepartment()))
                .filter(e -> e.getSalary() > 100000)
                .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
                .limit(5)
                .collect(Collectors.toList());

        // 12. Pagination: page 2, page size 2
        int page = 2;
        int pageSize = 2;

        List<Employee> pageTwo = employees.stream()
                .sorted(Comparator.comparingInt(Employee::getId))
                .skip((long) (page - 1) * pageSize)
                .limit(pageSize)
                .collect(Collectors.toList());

        // 13. Multiple sorting criteria
        List<Employee> multiSorted = employees.stream()
                .sorted(Comparator.comparing(Employee::getDepartment)
                        .thenComparing(Comparator.comparingInt(Employee::getSalary).reversed())
                        .thenComparing(Employee::getName))
                .collect(Collectors.toList());

        // 14. Infinite stream + limit
        List<Integer> generated = Stream.iterate(1, n -> n + 1)
                .limit(10)
                .collect(Collectors.toList());

        // 15. Infinite stream + skip + limit
        List<Integer> window = Stream.iterate(1, n -> n + 1)
                .skip(5)
                .limit(5)
                .collect(Collectors.toList());

        System.out.println("Ascending: " + ascending);
        System.out.println("Descending: " + descending);
        System.out.println("Unique: " + unique);
        System.out.println("First 3: " + firstThree);
        System.out.println("After first 3: " + afterThree);
        System.out.println("Top 3 distinct numbers: " + topThreeDistinct);
        System.out.println("Salary DESC: " + salaryDesc);
        System.out.println("Top 3 employees: " + topThreeEmployees);
        System.out.println("Top 3 distinct salaries: " + topThreeSalaries);
        System.out.println("Rank 4-6: " + rankFourToSix);
        System.out.println("Top 5 IT > 10 LPA: " + topFiveIT);
        System.out.println("Page 2: " + pageTwo);
        System.out.println("Multi-sort: " + multiSorted);
        System.out.println("Generated: " + generated);
        System.out.println("Window: " + window);
    }
}
```

---

# 26. Output Prediction 🔥

What is the output?

```java
List<Integer> result = Arrays.asList(5, 1, 5, 3, 2, 3, 4)
        .stream()
        .distinct()
        .sorted()
        .limit(3)
        .collect(Collectors.toList());
```

Answer:

```text
[1, 2, 3]
```

---

# 27. Interview Trap — Change the Order

```java
List<Integer> result = Arrays.asList(1, 1, 2, 2, 3, 3)
        .stream()
        .limit(4)
        .distinct()
        .collect(Collectors.toList());
```

Answer:

```text
[1, 2]
```

But:

```java
.stream()
.distinct()
.limit(4)
```

gives:

```text
[1, 2, 3]
```

**Always reason about the pipeline from left to right.**

---

# 28. 20 Interview Questions 🔥🔥🔥

1. What does `sorted()` do?
2. What is natural ordering?
3. What is the difference between `sorted()` and `sorted(Comparator)`?
4. How do you sort an Employee list by salary descending?
5. How do you sort by multiple fields?
6. What does `distinct()` use to identify duplicates?
7. Why are `equals()` and `hashCode()` important for `distinct()`?
8. Is `distinct()` stateless or stateful?
9. Is `sorted()` stateless or stateful?
10. What does `limit(n)` do?
11. What does `skip(n)` do?
12. How can `skip()` and `limit()` implement pagination?
13. Why can operation ordering change the result?
14. How do you find top 3 salaries?
15. How do you find top 3 distinct salaries?
16. How do you find the second-highest salary using Streams?
17. How do you find ranks 6–10 by salary?
18. Can `limit()` terminate an infinite stream?
19. Why is Stream `skip()` not always ideal for database pagination?
20. Explain `filter → sorted → limit` in a real application.

---

# 29. Coding Challenges 🔥🔥🔥

### Challenge 1
Sort integers ascending.

### Challenge 2
Sort integers descending.

### Challenge 3
Remove duplicate integers.

### Challenge 4
Find first 5 elements.

### Challenge 5
Skip first 5 elements.

### Challenge 6
Find top 3 numbers.

### Challenge 7
Find top 3 distinct numbers.

### Challenge 8
Find second-highest salary.

### Challenge 9
Find third-highest distinct salary.

### Challenge 10 ⭐⭐⭐⭐⭐
Find top 5 IT employees earning more than 10 LPA.

### Challenge 11
Sort employees by department and then salary descending.

### Challenge 12
Return employees ranked 6–10 by salary.

### Challenge 13
Implement page 3 with page size 10 using `skip()` and `limit()`.

### Challenge 14
Explain the result difference between `distinct().limit(3)` and `limit(3).distinct()`.

### Challenge 15 — 5-Year Interview Level ⭐⭐⭐⭐⭐
Create a pipeline that filters active employees, sorts by salary descending, removes duplicate business identities, and returns the requested page.

---

# 30. Common Mistakes ❌

### ❌ Mistake 1
Using `limit()` before `sorted()` when you actually need the global top N.

```java
// Not top 3 globally
.stream()
.limit(3)
.sorted(...)
```

Correct:

```java
.stream()
.sorted(...)
.limit(3)
```

### ❌ Mistake 2
Assuming `distinct()` automatically understands business identity for custom objects.

It depends on equality semantics.

### ❌ Mistake 3
Assuming `skip()` is efficient database pagination.

It operates on the Stream source; it does not automatically translate into database `OFFSET` behavior.

### ❌ Mistake 4
Ignoring pipeline order.

```text
distinct → limit
```
can produce a different result from:

```text
limit → distinct
```

### ❌ Mistake 5
Using a Stream over a huge database result merely to paginate data in memory.

Prefer database-side pagination when appropriate.

---

# 31. Final Revision Sheet 🧠

```text
sorted()
────────────────────────
Natural/custom ordering
Stateful intermediate operation
```

```text
distinct()
────────────────────────
Remove duplicates
Uses equality semantics
Stateful intermediate operation
```

```text
limit(n)
────────────────────────
Keep first n
Useful for Top-N and bounded streams
```

```text
skip(n)
────────────────────────
Ignore first n
Useful with limit() for windows/pages
```

### Golden Rules

```text
Need ordering?       → sorted()
Need unique values?  → distinct()
Need first N?        → limit(N)
Need after first N?  → skip(N)
Need page?           → skip(offset).limit(size)
```

### Top-N Pattern

```java
.stream()
.sorted(Comparator.comparingInt(Employee::getSalary).reversed())
.limit(3)
```

### Pagination Pattern

```java
.stream()
.skip((long) (page - 1) * pageSize)
.limit(pageSize)
```

---

# 32. 2-Minute Interview Script 🎤

> “`sorted`, `distinct`, `limit`, and `skip` are commonly used Stream intermediate operations. `sorted()` orders elements using natural ordering or a Comparator. `distinct()` removes duplicates using equality semantics, so for custom objects `equals()` and `hashCode()` are important. `limit(n)` keeps the first n elements and `skip(n)` ignores the first n. Combining skip and limit is useful for stream windows and simple in-memory pagination. Operation order matters: for example, `distinct().limit(3)` can produce a different result from `limit(3).distinct()`. For top-N problems, I normally sort first and then limit. For database-backed applications, I would generally prefer database-side pagination rather than loading a large dataset and applying Stream skip/limit in memory.”

---

# 🧪 Complete Practice Code

[GitHub — 9.13 Stream sorted/distinct/limit/skip Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/13-Stream-sorted-distinct-limit-skip)

---

## Navigation

[← 9.12 — Stream filter/map/flatMap Deep Dive](../12-Stream-filter-map-flatMap-Deep-Dive/README.md)

**Current → 9.13 — Stream `sorted()` / `distinct()` / `limit()` / `skip()` → ✅ Completed**

**Next → 9.14 — Stream `reduce()` / `collect()` Fundamentals**