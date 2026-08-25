# 9.18 — `Collectors.summarizingInt()` / `averagingInt()` / `summingInt()` Deep Dive

## 🎯 Interview Goal

Master Java Stream numeric collectors and know when to use each one:

```text
summingInt()      → total
averagingInt()    → average
summarizingInt()  → count + sum + min + average + max
```

These collectors are extremely common in employee, transaction, order and product-based interview problems.

---

# 1. `summingInt()` Fundamentals ⭐⭐⭐⭐⭐

Calculate total salary:

```java
int totalSalary = employees.stream()
        .collect(Collectors.summingInt(Employee::getSalary));
```

Mental model:

```text
Employee
   ↓
getSalary()
   ↓
summingInt()
   ↓
int total
```

Equivalent readable alternative:

```java
int totalSalary = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

### Interview line

> `summingInt()` is a Collector that sums an int-valued property, while `mapToInt(...).sum()` uses the primitive IntStream API directly.

---

# 2. `averagingInt()` Fundamentals ⭐⭐⭐⭐⭐

Calculate average salary:

```java
Double averageSalary = employees.stream()
        .collect(Collectors.averagingInt(Employee::getSalary));
```

Result type:

```text
Double
```

Even though salary is an `int`, the average is represented as `Double`.

Equivalent:

```java
OptionalDouble averageSalary = employees.stream()
        .mapToInt(Employee::getSalary)
        .average();
```

Important difference:

```text
averagingInt() → Double
IntStream.average() → OptionalDouble
```

---

# 3. `summarizingInt()` Fundamentals 🔥🔥🔥

Get multiple statistics in one pass:

```java
IntSummaryStatistics stats = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary));
```

Now:

```java
stats.getCount();
stats.getSum();
stats.getMin();
stats.getAverage();
stats.getMax();
```

Output conceptually:

```text
Count   → 6
Sum     → 780000
Min     → 90000
Average → 130000.0
Max     → 180000
```

---

# 4. `IntSummaryStatistics` ⭐⭐⭐⭐⭐

`Collectors.summarizingInt()` returns:

```java
IntSummaryStatistics
```

Important methods:

```java
getCount()
getSum()
getMin()
getAverage()
getMax()
```

Useful `toString()` output is also available for quick debugging:

```java
System.out.println(stats);
```

---

# 5. Why `summarizingInt()` Is Powerful

Suppose the interviewer asks for:

```text
count
sum
min
average
max
```

You could calculate each separately, but `summarizingInt()` packages all five statistics into one result object.

```java
IntSummaryStatistics stats = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary));
```

Then:

```java
long count = stats.getCount();
long sum = stats.getSum();
int min = stats.getMin();
double average = stats.getAverage();
int max = stats.getMax();
```

### Interview line

> When I need several integer statistics from the same stream, `summarizingInt()` is convenient because it returns count, sum, min, average and max together.

---

# 6. `summingInt()` vs `mapToInt().sum()` 🔥

### Collector approach

```java
int sum = employees.stream()
        .collect(Collectors.summingInt(Employee::getSalary));
```

### Primitive stream approach

```java
int sum = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

Both calculate a sum, but the APIs are different.

Use the primitive stream form when the operation naturally continues with other `IntStream` operations.

---

# 7. `averagingInt()` vs `mapToInt().average()` 🔥

### Collector

```java
Double average = employees.stream()
        .collect(Collectors.averagingInt(Employee::getSalary));
```

### Primitive stream

```java
OptionalDouble average = employees.stream()
        .mapToInt(Employee::getSalary)
        .average();
```

Important interview point:

```text
Collectors.averagingInt() → Double
IntStream.average()      → OptionalDouble
```

`OptionalDouble` makes the empty-stream case explicit.

---

# 8. `summarizingInt()` vs Separate Collectors 🔥🔥🔥

If you need only total:

```java
Collectors.summingInt(Employee::getSalary)
```

If you need only average:

```java
Collectors.averagingInt(Employee::getSalary)
```

If you need all standard integer statistics:

```java
Collectors.summarizingInt(Employee::getSalary)
```

Mental model:

```text
One metric       → dedicated collector
Many statistics  → summarizingInt()
```

---

# 9. `summarizingInt()` With `groupingBy()` 🔥🔥🔥

Department → salary statistics:

```java
Map<String, IntSummaryStatistics> statsByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

Now for each department:

```java
IntSummaryStatistics stats = statsByDepartment.get("IT");

System.out.println(stats.getCount());
System.out.println(stats.getSum());
System.out.println(stats.getMin());
System.out.println(stats.getAverage());
System.out.println(stats.getMax());
```

This is a very strong 5-year interview pattern.

---

# 10. `groupingBy()` + `summingInt()`

Department → total salary:

```java
Map<String, Integer> totalSalaryByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summingInt(Employee::getSalary)
        ));
```

---

# 11. `groupingBy()` + `averagingInt()`

Department → average salary:

```java
Map<String, Double> averageSalaryByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

---

# 12. `groupingBy()` + `summarizingInt()` ⭐⭐⭐⭐⭐

```java
Map<String, IntSummaryStatistics> salaryStatsByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

This gives every department:

```text
count
sum
min
average
max
```

without creating five separate maps.

---

# 13. `partitioningBy()` + `summarizingInt()`

Split employees into two groups and calculate salary statistics:

```java
Map<Boolean, IntSummaryStatistics> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() >= 100000,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

Conceptually:

```text
true  → salary >= 100000 statistics
false → salary < 100000 statistics
```

---

# 14. Nested `groupingBy()` + `summarizingInt()`

Department → Location → salary statistics:

```java
Map<String, Map<String, IntSummaryStatistics>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.groupingBy(
                        Employee::getLocation,
                        Collectors.summarizingInt(Employee::getSalary)
                )
        ));
```

This pattern is useful when multiple business dimensions are involved.

---

# 15. `summarizingLong()` / `summarizingDouble()` 🔥

The same family exists for other numeric types:

```java
Collectors.summarizingInt(...)
Collectors.summarizingLong(...)
Collectors.summarizingDouble(...)
```

Return types:

```text
summarizingInt    → IntSummaryStatistics
summarizingLong   → LongSummaryStatistics
summarizingDouble → DoubleSummaryStatistics
```

---

# 16. Numeric Collector Type Cheat Sheet

| Requirement | Collector | Result |
|---|---|---|
| Sum int values | `summingInt()` | `Integer` |
| Average int values | `averagingInt()` | `Double` |
| All int statistics | `summarizingInt()` | `IntSummaryStatistics` |
| Sum long values | `summingLong()` | `Long` |
| Average long values | `averagingLong()` | `Double` |
| All long statistics | `summarizingLong()` | `LongSummaryStatistics` |
| Sum double values | `summingDouble()` | `Double` |
| Average double values | `averagingDouble()` | `Double` |
| All double statistics | `summarizingDouble()` | `DoubleSummaryStatistics` |

---

# 17. `summarizingInt()` Empty Stream ⚠️

```java
IntSummaryStatistics stats = Stream.<Integer>empty()
        .collect(Collectors.summarizingInt(Integer::intValue));
```

For an empty statistics object:

```text
count   = 0
sum     = 0
average = 0.0
min     = Integer.MAX_VALUE
max     = Integer.MIN_VALUE
```

Do not interpret the default min/max values as actual data values when count is zero.

Always check:

```java
if (stats.getCount() > 0) {
    System.out.println(stats.getMin());
}
```

---

# 18. `summingInt()` Empty Stream

```java
int sum = Stream.<Integer>empty()
        .collect(Collectors.summingInt(Integer::intValue));
```

Result:

```text
0
```

---

# 19. `averagingInt()` Empty Stream

```java
Double average = Stream.<Integer>empty()
        .collect(Collectors.averagingInt(Integer::intValue));
```

Result:

```text
0.0
```

This differs from:

```java
IntStream.empty().average()
```

which returns:

```java
OptionalDouble.empty()
```

---

# 20. Avoiding Integer Overflow 🔥

Be careful with `summingInt()` when values can exceed the `int` range.

For large numeric values, use an appropriate wider type:

```java
long total = employees.stream()
        .collect(Collectors.summingLong(Employee::getSalary));
```

or:

```java
long total = employees.stream()
        .mapToLong(Employee::getSalary)
        .sum();
```

### Interview line

> I choose the numeric type based on the maximum possible aggregate, not merely the field's current type.

---

# 21. `summarizingInt()` With Filtering 🔥

Calculate statistics only for IT employees:

```java
IntSummaryStatistics stats = employees.stream()
        .filter(e -> e.getDepartment().equals("IT"))
        .collect(Collectors.summarizingInt(Employee::getSalary));
```

---

# 22. `groupingBy()` + `filter()` + `summarizingInt()`

For each department, calculate statistics only for employees earning at least 100000:

```java
Map<String, IntSummaryStatistics> result = employees.stream()
        .filter(e -> e.getSalary() >= 100000)
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

Important distinction:

```text
stream.filter() before grouping
→ removes records globally before grouping
```

If you need to retain every department and represent empty downstream results, use an appropriate downstream `filtering()` collector instead.

---

# 23. Java 9 `Collectors.filtering()` + `summarizingInt()`

```java
Map<String, IntSummaryStatistics> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.summarizingInt(Employee::getSalary)
                )
        ));
```

This keeps the grouping structure while filtering inside each group.

---

# 24. Finding Min / Max From Statistics

```java
IntSummaryStatistics stats = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary));

int minimum = stats.getMin();
int maximum = stats.getMax();
```

If you only need min/max and the object itself is required, use `min()` / `max()` with a Comparator instead:

```java
Optional<Employee> highestPaid = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary));
```

### Important distinction

```text
Need numeric min/max       → summarizingInt()
Need actual Employee       → max()/min()
```

---

# 25. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class NumericCollectorsDemo {

    static class Employee {
        private final int id;
        private final String name;
        private final String department;
        private final String location;
        private final int salary;

        Employee(int id, String name, String department,
                 String location, int salary) {
            this.id = id;
            this.name = name;
            this.department = department;
            this.location = location;
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

        public String getLocation() {
            return location;
        }

        public int getSalary() {
            return salary;
        }

        @Override
        public String toString() {
            return id + " - " + name + " - " + department
                    + " - " + location + " - " + salary;
        }
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", "Bangalore", 150000),
                new Employee(2, "Rahul", "IT", "Bangalore", 120000),
                new Employee(3, "Amit", "IT", "Pune", 180000),
                new Employee(4, "Priya", "HR", "Bangalore", 130000),
                new Employee(5, "Ravi", "HR", "Pune", 90000),
                new Employee(6, "Sneha", "Finance", "Bangalore", 110000)
        );

        // 1. Total salary
        int totalSalary = employees.stream()
                .collect(Collectors.summingInt(Employee::getSalary));

        // 2. Average salary
        Double averageSalary = employees.stream()
                .collect(Collectors.averagingInt(Employee::getSalary));

        // 3. Complete salary statistics
        IntSummaryStatistics salaryStats = employees.stream()
                .collect(Collectors.summarizingInt(Employee::getSalary));

        // 4. Total salary by department
        Map<String, Integer> totalByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.summingInt(Employee::getSalary)
                ));

        // 5. Average salary by department
        Map<String, Double> averageByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.averagingInt(Employee::getSalary)
                ));

        // 6. Complete statistics by department
        Map<String, IntSummaryStatistics> statsByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.summarizingInt(Employee::getSalary)
                ));

        // 7. Partition by salary and calculate statistics
        Map<Boolean, IntSummaryStatistics> partitionedStats = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() >= 100000,
                        Collectors.summarizingInt(Employee::getSalary)
                ));

        // 8. IT statistics only
        IntSummaryStatistics itStats = employees.stream()
                .filter(e -> e.getDepartment().equals("IT"))
                .collect(Collectors.summarizingInt(Employee::getSalary));

        // 9. Statistics after salary filter
        Map<String, IntSummaryStatistics> filteredStats = employees.stream()
                .filter(e -> e.getSalary() >= 100000)
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.summarizingInt(Employee::getSalary)
                ));

        // 10. Statistics per department and location
        Map<String, Map<String, IntSummaryStatistics>> locationStats = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.groupingBy(
                                Employee::getLocation,
                                Collectors.summarizingInt(Employee::getSalary)
                        )
                ));

        // 11. Primitive stream equivalents
        int primitiveSum = employees.stream()
                .mapToInt(Employee::getSalary)
                .sum();

        OptionalDouble primitiveAverage = employees.stream()
                .mapToInt(Employee::getSalary)
                .average();

        IntSummaryStatistics primitiveStats = employees.stream()
                .mapToInt(Employee::getSalary)
                .summaryStatistics();

        // 12. Empty statistics
        IntSummaryStatistics emptyStats = Stream.<Integer>empty()
                .collect(Collectors.summarizingInt(Integer::intValue));

        System.out.println("Total salary: " + totalSalary);
        System.out.println("Average salary: " + averageSalary);
        System.out.println("Salary statistics: " + salaryStats);
        System.out.println("Count: " + salaryStats.getCount());
        System.out.println("Sum: " + salaryStats.getSum());
        System.out.println("Min: " + salaryStats.getMin());
        System.out.println("Average: " + salaryStats.getAverage());
        System.out.println("Max: " + salaryStats.getMax());
        System.out.println("Total by department: " + totalByDepartment);
        System.out.println("Average by department: " + averageByDepartment);
        System.out.println("Stats by department: " + statsByDepartment);
        System.out.println("Partitioned stats: " + partitionedStats);
        System.out.println("IT stats: " + itStats);
        System.out.println("Filtered stats: " + filteredStats);
        System.out.println("Location stats: " + locationStats);
        System.out.println("Primitive sum: " + primitiveSum);
        System.out.println("Primitive average: " + primitiveAverage);
        System.out.println("Primitive stats: " + primitiveStats);
        System.out.println("Empty stats: " + emptyStats);
    }
}
```

---

# 26. Interview Scenario — Department Salary Dashboard 🔥🔥🔥

**Question:** For every department return count, total salary, minimum salary, average salary and maximum salary.

### Best answer

```java
Map<String, IntSummaryStatistics> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

Then:

```java
IntSummaryStatistics stats = result.get("IT");

stats.getCount();
stats.getSum();
stats.getMin();
stats.getAverage();
stats.getMax();
```

### Why this is strong

Instead of maintaining:

```text
Map<Department, Count>
Map<Department, Sum>
Map<Department, Min>
Map<Department, Average>
Map<Department, Max>
```

you have:

```text
Map<Department, IntSummaryStatistics>
```

---

# 27. Interview Scenario — Total Transaction Amount

```java
long total = transactions.stream()
        .collect(Collectors.summingLong(Transaction::getAmount));
```

For monetary/business values, choose the numeric representation carefully; for exact financial calculations, `BigDecimal` is often preferable to floating-point types.

---

# 28. Interview Scenario — Average Product Price

```java
Double averagePrice = products.stream()
        .collect(Collectors.averagingInt(Product::getPrice));
```

If price is a `double`:

```java
Double averagePrice = products.stream()
        .collect(Collectors.averagingDouble(Product::getPrice));
```

---

# 29. Interview Scenario — Product Price Statistics

```java
DoubleSummaryStatistics stats = products.stream()
        .collect(Collectors.summarizingDouble(Product::getPrice));
```

Then:

```java
stats.getCount();
stats.getSum();
stats.getMin();
stats.getAverage();
stats.getMax();
```

---

# 30. Interview Scenario — Highest Numeric Salary vs Highest-Paid Employee

### Need salary number

```java
int maxSalary = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary))
        .getMax();
```

### Need Employee object

```java
Optional<Employee> employee = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary));
```

### Interview line

> If I only need numeric statistics, `summarizingInt()` is convenient. If I need the actual employee object, I use `max()` or `min()` with a Comparator.

---

# 31. Interview Scenario — Count High Earners Per Department

Simple approach:

```java
Map<String, Long> result = employees.stream()
        .filter(e -> e.getSalary() >= 100000)
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

This is different from `summarizingInt()` because the requirement is only a count.

---

# 32. 25 Interview Questions 🎯

1. What is `Collectors.summingInt()`?
2. What does `averagingInt()` return?
3. What does `summarizingInt()` return?
4. What methods are available on `IntSummaryStatistics`?
5. Difference between `summingInt()` and `mapToInt().sum()`?
6. Difference between `averagingInt()` and `IntStream.average()`?
7. Why does `IntStream.average()` return `OptionalDouble`?
8. Why does `averagingInt()` return `Double`?
9. What happens with an empty `summarizingInt()` stream?
10. What are min/max values for empty `IntSummaryStatistics`?
11. Why should you check `getCount()` before interpreting empty min/max?
12. When should you use `summarizingInt()` instead of `summingInt()`?
13. How do you calculate salary statistics per department?
14. How do you combine `groupingBy()` and `summarizingInt()`?
15. How do you combine `partitioningBy()` and `summarizingInt()`?
16. Difference between numeric maximum and maximum object?
17. When would `max()` be better than `summarizingInt()`?
18. What are `summarizingLong()` and `summarizingDouble()`?
19. What is `IntSummaryStatistics`?
20. How can numeric overflow happen with `summingInt()`?
21. When should `summingLong()` be preferred?
22. How do you calculate average salary per department?
23. How do you calculate total salary per department?
24. How do you preserve all departments while filtering downstream values?
25. Explain `groupingBy + summarizingInt` in 2 minutes.

---

# 33. Coding Challenges 💻

### Challenge 1
Calculate total employee salary using `summingInt()`.

### Challenge 2
Calculate average employee salary using `averagingInt()`.

### Challenge 3
Calculate count, sum, min, average and max using `summarizingInt()`.

### Challenge 4
Calculate total salary by department.

### Challenge 5
Calculate average salary by department.

### Challenge 6 ⭐⭐⭐⭐⭐
Calculate complete salary statistics by department.

### Challenge 7
Partition employees by salary >= 100000 and calculate statistics for each partition.

### Challenge 8
Calculate statistics only for IT employees.

### Challenge 9
Calculate salary statistics for employees earning >= 100000.

### Challenge 10
Calculate statistics by department and location.

### Challenge 11
Calculate total transaction amount using `summingLong()`.

### Challenge 12
Calculate average product price using `averagingDouble()`.

### Challenge 13
Calculate complete product price statistics using `summarizingDouble()`.

### Challenge 14
Find numeric maximum salary using `summarizingInt()`.

### Challenge 15
Find the actual highest-paid employee using `max()`.

### Challenge 16
Explain why these two solutions return different types:

```java
Collectors.averagingInt(...)
```

and

```java
mapToInt(...).average()
```

### Challenge 17
Calculate total salary per department using `summarizingInt()` and extract only `getSum()`.

### Challenge 18
Calculate maximum salary per department using `summarizingInt()`.

### Challenge 19
Calculate minimum salary per department using `summarizingInt()`.

### Challenge 20
Calculate average salary per department using `summarizingInt()` instead of `averagingInt()`.

### Challenge 21 — 5-Year Interview Level 🔥
Build a department salary dashboard containing:

```text
Department
Employee Count
Total Salary
Min Salary
Average Salary
Max Salary
```

using a single `Map<String, IntSummaryStatistics>`.

### Challenge 22
Build the same dashboard using `LongSummaryStatistics` for large salary values.

### Challenge 23
Partition transactions into high-value and low-value transactions and calculate statistics for both.

### Challenge 24
Explain when `summingInt()` may be unsafe because of numeric range and propose a safer type.

### Challenge 25 — Production Scenario 🔥🔥🔥
Given millions of transaction records, design a stream-based numeric aggregation and explain your choice between primitive streams and collectors, including numeric type and overflow considerations.

---

# 34. Common Mistakes ❌

### ❌ Mistake 1 — Using `summarizingInt()` when only sum is required

If only total is required:

```java
Collectors.summingInt(...)
```

is clearer.

### ❌ Mistake 2 — Confusing `Double` with `OptionalDouble`

```text
averagingInt()       → Double
IntStream.average()  → OptionalDouble
```

### ❌ Mistake 3 — Treating empty min/max as real values

For empty `IntSummaryStatistics`, min/max use sentinel values. Check `getCount()`.

### ❌ Mistake 4 — Ignoring integer overflow

If the aggregate can exceed the `int` range, use a wider numeric type.

### ❌ Mistake 5 — Using `summarizingInt()` when the actual object is required

Use:

```java
stream.max(Comparator.comparingInt(...))
```

when you need the object itself.

### ❌ Mistake 6 — Creating many maps for one dashboard

Prefer:

```java
Map<String, IntSummaryStatistics>
```

when the requirements naturally match summary statistics.

---

# 35. Final Revision Sheet 🧠

```text
summingInt()
────────────────────
Need total
Result → Integer
```

```text
averagingInt()
────────────────────
Need average
Result → Double
```

```text
summarizingInt()
────────────────────
Need count + sum + min + average + max
Result → IntSummaryStatistics
```

```text
groupingBy(key, summarizingInt(...))
────────────────────
Key → complete numeric statistics
```

### Golden Rules

```text
Only sum?                    → summingInt()
Only average?                → averagingInt()
All five standard metrics?   → summarizingInt()
Need actual max object?      → max(Comparator)
Need large integer totals?   → summingLong()
Need double statistics?       → summarizingDouble()
```

---

# 36. 2-Minute Interview Script 🎤

> “`summingInt()`, `averagingInt()` and `summarizingInt()` are numeric collectors. `summingInt()` calculates the total of an int-valued property, while `averagingInt()` calculates the average and returns a `Double`. `summarizingInt()` is useful when I need multiple statistics because it returns an `IntSummaryStatistics` containing count, sum, min, average and max. A common interview pattern is `groupingBy()` with `summarizingInt()` to build department-wise salary statistics in a single map. If I only need the highest-paid employee object rather than the numeric maximum salary, I prefer `max()` with a Comparator. I also consider numeric range: if an int sum can overflow, I use `summingLong()` or another appropriate representation. For empty streams, I pay attention to the different APIs: `averagingInt()` returns `0.0`, while `IntStream.average()` returns an empty `OptionalDouble`.”

---

# 🧪 Complete Practice Code

[GitHub — 9.18 `summarizingInt()` / `averagingInt()` / `summingInt()` Deep Dive Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/18-Collectors-summarizingInt-averagingInt-summingInt-Deep-Dive)

---

## Navigation

[← 9.17 — `joining()` / `mapping()` / `collectingAndThen()` Deep Dive](../17-Collectors-joining-mapping-collectingAndThen-Deep-Dive/README.md)

**Current → 9.18 — `summarizingInt()` / `averagingInt()` / `summingInt()` → ✅ Completed**

**Next → 9.19 — `Collectors.maxBy()` / `minBy()` / `counting()`**