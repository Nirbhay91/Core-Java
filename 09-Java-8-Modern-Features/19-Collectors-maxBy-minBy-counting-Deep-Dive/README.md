# 9.19 — `Collectors.maxBy()` / `minBy()` / `counting()` Deep Dive

## 🎯 Interview Goal

Master three important downstream collectors:

```text
maxBy()   → maximum element according to Comparator
minBy()   → minimum element according to Comparator
counting() → number of elements
```

These are especially important with `groupingBy()` and `partitioningBy()`.

---

# 1. `Collectors.maxBy()` Fundamentals ⭐⭐⭐⭐⭐

Find the employee with the highest salary:

```java
Optional<Employee> highestPaid = employees.stream()
        .collect(Collectors.maxBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

Result:

```java
Optional<Employee>
```

### Interview line

> `maxBy()` is a Collector that uses a Comparator to select the maximum element and returns it as an Optional.

---

# 2. `Collectors.minBy()` Fundamentals ⭐⭐⭐⭐⭐

Find the employee with the lowest salary:

```java
Optional<Employee> lowestPaid = employees.stream()
        .collect(Collectors.minBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

Result:

```java
Optional<Employee>
```

---

# 3. Why Does `maxBy()` Return `Optional`? 🔥

A stream may be empty:

```java
Optional<Employee> result = Stream.<Employee>empty()
        .collect(Collectors.maxBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

Result:

```text
Optional.empty()
```

So you should handle it safely:

```java
highestPaid.ifPresent(System.out::println);
```

Or:

```java
Employee employee = highestPaid.orElse(null);
```

---

# 4. `counting()` Fundamentals ⭐⭐⭐⭐⭐

Count all employees:

```java
Long count = employees.stream()
        .collect(Collectors.counting());
```

Result type:

```text
Long
```

Equivalent:

```java
long count = employees.stream().count();
```

### Interview line

> `counting()` is a Collector that counts stream elements and returns a `Long`.

---

# 5. `counting()` With `groupingBy()` 🔥🔥🔥

Count employees in each department:

```java
Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Output conceptually:

```text
IT       → 3
HR       → 2
Finance  → 1
```

This is one of the most common interview patterns.

---

# 6. `maxBy()` With `groupingBy()` 🔥🔥🔥

Find the highest-paid employee in every department:

```java
Map<String, Optional<Employee>> highestPaidByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

Mental model:

```text
Employee
   ↓
groupingBy(department)
   ↓
maxBy(salary)
   ↓
Department → Optional<Employee>
```

---

# 7. `minBy()` With `groupingBy()` 🔥🔥🔥

Find the lowest-paid employee in every department:

```java
Map<String, Optional<Employee>> lowestPaidByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.minBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

---

# 8. `counting()` vs `summarizingInt()` ⭐⭐⭐⭐⭐

If only count is needed:

```java
Collectors.counting()
```

If you need count + sum + min + average + max:

```java
Collectors.summarizingInt(Employee::getSalary)
```

Mental model:

```text
Only count       → counting()
Numeric summary  → summarizingInt()
Actual max object → maxBy()/max()
```

---

# 9. `maxBy()` vs Stream `max()` 🔥🔥🔥

### Stream operation

```java
Optional<Employee> highestPaid = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary));
```

### Collector

```java
Optional<Employee> highestPaid = employees.stream()
        .collect(Collectors.maxBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

For a simple whole-stream maximum, `stream.max()` is generally more direct.

`Collectors.maxBy()` becomes particularly useful as a **downstream collector**:

```java
Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingInt(Employee::getSalary))
)
```

### Interview line

> I prefer `Stream.max()` for a top-level maximum, while `Collectors.maxBy()` is especially useful when maximum selection is part of a downstream collection operation.

---

# 10. `minBy()` vs Stream `min()`

### Stream operation

```java
Optional<Employee> lowestPaid = employees.stream()
        .min(Comparator.comparingInt(Employee::getSalary));
```

### Collector

```java
Optional<Employee> lowestPaid = employees.stream()
        .collect(Collectors.minBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

Same conceptual selection; different API context.

---

# 11. `groupingBy()` + `counting()` + `maxBy()` 🔥🔥🔥

You can combine multiple requirements in a department-level result, although each metric may need a separate downstream structure:

```java
Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));

Map<String, Optional<Employee>> maxByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(Comparator.comparingInt(Employee::getSalary))
        ));
```

For a single dashboard containing many statistics, `summarizingInt()` may be more suitable for numeric salary statistics.

---

# 12. `partitioningBy()` + `counting()`

Count high and low earners:

```java
Map<Boolean, Long> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() >= 100000,
                Collectors.counting()
        ));
```

Conceptually:

```text
true  → count of employees with salary >= 100000
false → count of employees with salary < 100000
```

---

# 13. `partitioningBy()` + `maxBy()`

Find the highest-paid employee in each salary partition:

```java
Map<Boolean, Optional<Employee>> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() >= 100000,
                Collectors.maxBy(Comparator.comparingInt(Employee::getSalary))
        ));
```

---

# 14. `partitioningBy()` + `minBy()`

```java
Map<Boolean, Optional<Employee>> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() >= 100000,
                Collectors.minBy(Comparator.comparingInt(Employee::getSalary))
        ));
```

---

# 15. `maxBy()` With Multiple Sort Keys 🔥

Highest salary; if salaries tie, choose the employee with the highest ID:

```java
Comparator<Employee> comparator = Comparator
        .comparingInt(Employee::getSalary)
        .thenComparingInt(Employee::getId);

Optional<Employee> result = employees.stream()
        .collect(Collectors.maxBy(comparator));
```

This is a strong interview-level variation.

---

# 16. `minBy()` With Multiple Sort Keys

```java
Comparator<Employee> comparator = Comparator
        .comparingInt(Employee::getSalary)
        .thenComparing(Employee::getName);

Optional<Employee> result = employees.stream()
        .collect(Collectors.minBy(comparator));
```

---

# 17. Highest Paid Employee Per Department — Most Common Interview Question ⭐⭐⭐⭐⭐

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

If you want to remove the `Optional` after grouping, `collectingAndThen()` can be used carefully:

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        ),
                        Optional::orElseThrow
                )
        ));
```

This assumes every group is non-empty, which is true for groups actually created by `groupingBy()`.

---

# 18. Lowest Paid Employee Per Department

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.minBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

---

# 19. Count Employees Per Department

```java
Map<String, Long> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

---

# 20. Count Employees Per Location

```java
Map<String, Long> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getLocation,
                Collectors.counting()
        ));
```

---

# 21. Count Employees With Salary > 100000

```java
long count = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .collect(Collectors.counting());
```

For a top-level count, this is simpler:

```java
long count = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .count();
```

Again, `counting()` is especially valuable as a downstream collector.

---

# 22. Department → Highest Salary Number

If you need the numeric maximum rather than the employee object:

```java
Map<String, IntSummaryStatistics> stats = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingInt(Employee::getSalary)
        ));

Map<String, Integer> maxSalaryByDepartment = stats.entrySet().stream()
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> e.getValue().getMax()
        ));
```

Or use `maxBy()` if the employee object is required.

---

# 23. `maxBy()` vs `summarizingInt().getMax()` 🔥

### Need the Employee

```java
Optional<Employee> maxEmployee = employees.stream()
        .collect(Collectors.maxBy(
                Comparator.comparingInt(Employee::getSalary)
        ));
```

### Need only salary

```java
int maxSalary = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary))
        .getMax();
```

This distinction is frequently tested.

---

# 24. `counting()` Return Type ⚠️

`Collectors.counting()` returns:

```java
Long
```

Therefore:

```java
Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Do not incorrectly declare:

```java
Map<String, Integer> result;
```

---

# 25. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;

public class MaxByMinByCountingDemo {

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

        // 1. Highest-paid employee in the whole list
        Optional<Employee> highestPaid = employees.stream()
                .collect(Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                ));

        // 2. Lowest-paid employee in the whole list
        Optional<Employee> lowestPaid = employees.stream()
                .collect(Collectors.minBy(
                        Comparator.comparingInt(Employee::getSalary)
                ));

        // 3. Count all employees
        Long employeeCount = employees.stream()
                .collect(Collectors.counting());

        // 4. Employee count by department
        Map<String, Long> countByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.counting()
                ));

        // 5. Highest-paid employee by department
        Map<String, Optional<Employee>> highestPaidByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 6. Lowest-paid employee by department
        Map<String, Optional<Employee>> lowestPaidByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.minBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 7. Count by location
        Map<String, Long> countByLocation = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getLocation,
                        Collectors.counting()
                ));

        // 8. High-earner count by department
        Map<String, Long> highEarnersByDepartment = employees.stream()
                .filter(e -> e.getSalary() >= 100000)
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.counting()
                ));

        // 9. Partition employees and count each partition
        Map<Boolean, Long> partitionedCount = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() >= 100000,
                        Collectors.counting()
                ));

        // 10. Highest-paid employee in each salary partition
        Map<Boolean, Optional<Employee>> partitionedMax = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() >= 100000,
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 11. Lowest-paid employee in each salary partition
        Map<Boolean, Optional<Employee>> partitionedMin = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() >= 100000,
                        Collectors.minBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 12. Tie-breaker: highest salary, then highest ID
        Comparator<Employee> salaryThenId = Comparator
                .comparingInt(Employee::getSalary)
                .thenComparingInt(Employee::getId);

        Optional<Employee> highestWithTieBreaker = employees.stream()
                .collect(Collectors.maxBy(salaryThenId));

        // 13. Highest-paid employee by department without Optional in final map
        Map<String, Employee> highestPaidWithoutOptional = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.collectingAndThen(
                                Collectors.maxBy(
                                        Comparator.comparingInt(Employee::getSalary)
                                ),
                                Optional::orElseThrow
                        )
                ));

        // 14. Empty maxBy example
        Optional<Employee> emptyMax = Collections.<Employee>emptyList()
                .stream()
                .collect(Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                ));

        System.out.println("Highest paid: " + highestPaid);
        System.out.println("Lowest paid: " + lowestPaid);
        System.out.println("Employee count: " + employeeCount);
        System.out.println("Count by department: " + countByDepartment);
        System.out.println("Highest paid by department: " + highestPaidByDepartment);
        System.out.println("Lowest paid by department: " + lowestPaidByDepartment);
        System.out.println("Count by location: " + countByLocation);
        System.out.println("High earners by department: " + highEarnersByDepartment);
        System.out.println("Partitioned count: " + partitionedCount);
        System.out.println("Partitioned max: " + partitionedMax);
        System.out.println("Partitioned min: " + partitionedMin);
        System.out.println("Highest with tie-breaker: " + highestWithTieBreaker);
        System.out.println("Highest without Optional: " + highestPaidWithoutOptional);
        System.out.println("Empty max: " + emptyMax);
    }
}
```

---

# 26. Interview Scenario — Highest Paid Employee Per Department 🔥🔥🔥

**Question:** Find the highest-paid employee from each department.

Best downstream collector solution:

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

### 2-minute explanation

```text
groupingBy()      → creates one group per department
maxBy()            → selects maximum Employee inside each group
Comparator         → tells maxBy() how salary should be compared
Optional<Employee> → handles the possibility of no maximum
```

---

# 27. Interview Scenario — Lowest Paid Employee Per Department

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.minBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

---

# 28. Interview Scenario — Employee Count Per Department

```java
Map<String, Long> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

### Interview line

> `counting()` is especially useful as a downstream collector because it lets me count elements independently inside each group.

---

# 29. Interview Scenario — Department With Most Employees 🔥

```java
Optional<Map.Entry<String, Long>> department = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ))
        .entrySet()
        .stream()
        .max(Map.Entry.comparingByValue());
```

If you need the department name:

```java
String departmentName = department
        .map(Map.Entry::getKey)
        .orElse(null);
```

---

# 30. Interview Scenario — Department With Highest Average Salary

First calculate average salary per department:

```java
Map<String, Double> averageByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

Then select the department with the highest average:

```java
Optional<Map.Entry<String, Double>> result = averageByDepartment.entrySet()
        .stream()
        .max(Map.Entry.comparingByValue());
```

This is different from finding the department containing the highest-paid individual employee.

---

# 31. Important Comparison Table 🧠

| Requirement | Best API |
|---|---|
| Whole-stream max object | `stream.max()` |
| Whole-stream min object | `stream.min()` |
| Downstream max object | `Collectors.maxBy()` |
| Downstream min object | `Collectors.minBy()` |
| Whole-stream count | `stream.count()` |
| Downstream count | `Collectors.counting()` |
| Numeric max | `summarizingInt().getMax()` / `mapToInt().max()` |
| Numeric min | `summarizingInt().getMin()` / `mapToInt().min()` |

---

# 32. 25 Interview Questions 🎯

1. What is `Collectors.maxBy()`?
2. What is `Collectors.minBy()`?
3. Why do `maxBy()` and `minBy()` return `Optional`?
4. What is `Collectors.counting()`?
5. What is the return type of `counting()`?
6. Difference between `stream.max()` and `Collectors.maxBy()`?
7. When is `maxBy()` especially useful?
8. How do you find the highest-paid employee per department?
9. How do you find the lowest-paid employee per department?
10. How do you count employees per department?
11. How do you combine `groupingBy()` and `maxBy()`?
12. How do you combine `groupingBy()` and `minBy()`?
13. How do you combine `groupingBy()` and `counting()`?
14. How do you use `maxBy()` with a custom Comparator?
15. How do you implement a tie-breaker?
16. Difference between highest salary and highest-paid employee?
17. Why might `summarizingInt()` be better than `maxBy()` for numeric statistics?
18. How do you partition employees and count each partition?
19. How do you find the maximum employee per partition?
20. How do you remove `Optional` from a `groupingBy + maxBy` result?
21. Is `counting()` preferable to `count()` for a top-level stream?
22. What happens when `maxBy()` receives an empty stream?
23. How do you find the department with the most employees?
24. How do you find the department with the highest average salary?
25. Explain `groupingBy + maxBy` in two minutes.

---

# 33. Coding Challenges 💻

1. Find the highest-paid employee.
2. Find the lowest-paid employee.
3. Count all employees.
4. Count employees by department.
5. Count employees by location.
6. Find highest-paid employee per department.
7. Find lowest-paid employee per department.
8. Find highest-paid employee per location.
9. Find lowest-paid employee per location.
10. Partition employees by salary and count both groups.
11. Partition employees by salary and find the maximum in each group.
12. Partition employees by salary and find the minimum in each group.
13. Add salary as the primary comparator and employee ID as tie-breaker.
14. Find department with most employees.
15. Find department with fewest employees.
16. Find department with highest average salary.
17. Find department with lowest average salary.
18. Return highest-paid employee per department without `Optional`.
19. Return lowest-paid employee per department without `Optional`.
20. Compare `maxBy()` with `summarizingInt().getMax()`.
21. Find highest salary and employee name for each department.
22. Find employee count and highest-paid employee per department using separate downstream results.
23. Find the most populated location.
24. Find the highest-paid employee among employees earning at least 100000.
25. **5-year level:** Explain the trade-offs between `maxBy()`, `stream.max()`, `summarizingInt()` and sorting when finding a maximum.

---

# 34. Common Mistakes ❌

### ❌ Mistake 1 — Forgetting `Optional`

```java
Optional<Employee> result = employees.stream()
        .collect(Collectors.maxBy(...));
```

Do not assume the result is directly an `Employee`.

### ❌ Mistake 2 — Using `maxBy()` when only a number is required

If only maximum salary is needed, numeric aggregation may be clearer:

```java
int maxSalary = employees.stream()
        .collect(Collectors.summarizingInt(Employee::getSalary))
        .getMax();
```

### ❌ Mistake 3 — Using `counting()` when top-level `.count()` is simpler

```java
long count = employees.stream().count();
```

Use `counting()` mainly when it is a downstream collector.

### ❌ Mistake 4 — Confusing highest salary with highest-paid employee

```text
highest salary       → int
highest-paid employee → Employee
```

### ❌ Mistake 5 — Ignoring ties

If multiple employees have the same salary, define a tie-breaker when business requirements need deterministic selection.

---

# 35. Final Revision Sheet 🧠

```text
maxBy(Comparator)
────────────────────────
Maximum element
Result → Optional<T>
```

```text
minBy(Comparator)
────────────────────────
Minimum element
Result → Optional<T>
```

```text
counting()
────────────────────────
Number of elements
Result → Long
```

### Golden Rules

```text
Whole stream max object   → stream.max()
Whole stream min object   → stream.min()
Grouped max object        → maxBy()
Grouped min object        → minBy()
Grouped count             → counting()
Numeric max only          → summarizingInt().getMax()
```

### Most important pattern

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

---

# 36. 2-Minute Interview Script 🎤

> “`Collectors.maxBy()` and `minBy()` are downstream collectors that use a Comparator to select the maximum or minimum element and return an Optional. They become especially useful with `groupingBy()`, for example to find the highest-paid employee in every department. `Collectors.counting()` counts elements and returns a Long, and it is particularly useful as a downstream collector to count employees per department. For a simple whole-stream maximum I generally prefer `stream.max()`, while `maxBy()` is useful when maximum selection is part of a larger collector pipeline. If I only need a numeric maximum rather than the employee object, `summarizingInt()` or a primitive stream can be more appropriate.”

---

# 🧪 Complete Practice Code

[GitHub — 9.19 `maxBy()` / `minBy()` / `counting()` Deep Dive Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/19-Collectors-maxBy-minBy-counting-Deep-Dive)

---

## Navigation

[← 9.18 — `summarizingInt()` / `averagingInt()` / `summingInt()` Deep Dive](../18-Collectors-summarizingInt-averagingInt-summingInt-Deep-Dive/README.md)

**Current → 9.19 — `maxBy()` / `minBy()` / `counting()` → ✅ Completed**

**Next → 9.20 — `Collectors.filtering()` / `flatMapping()` / `teeing()`**