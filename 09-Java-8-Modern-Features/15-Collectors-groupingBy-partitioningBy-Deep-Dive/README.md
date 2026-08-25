# 9.15 — `Collectors.groupingBy()` / `partitioningBy()` Deep Dive

## 🎯 Interview Goal

Master how to transform a Stream into grouped, counted, mapped and partitioned results.

```text
groupingBy()      → key → group/aggregation
partitioningBy()  → true/false → two groups
```

---

# 1. `groupingBy()` Fundamentals ⭐⭐⭐⭐⭐

Group employees by department:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Mental model:

```text
Employee Stream
      ↓
groupingBy(department)
      ↓
Department → List<Employee>
```

---

# 2. Why `groupingBy()` Is Important

It is one of the most common Java Stream interview patterns.

Typical requirements:

```text
Employees by department
Employees by city
Employees by role
Orders by status
Products by category
Transactions by type
```

---

# 3. `groupingBy()` Signature ⭐⭐⭐⭐⭐

Conceptually, the common overloads are:

```java
Collectors.groupingBy(classifier)
```

```java
Collectors.groupingBy(classifier, downstreamCollector)
```

```java
Collectors.groupingBy(classifier, mapFactory, downstreamCollector)
```

The classifier decides the group key.

---

# 4. Grouping by Department

```java
Map<String, List<Employee>> byDepartment = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Example:

```text
IT      → [Nirbhay, Rahul, Amit]
HR      → [Priya, Ravi]
Finance → [Sneha]
```

---

# 5. Grouping by City

```java
Map<String, List<Employee>> byCity = employees.stream()
        .collect(Collectors.groupingBy(Employee::getCity));
```

Same collector pattern; only the classifier changes.

---

# 6. `groupingBy()` + `counting()` 🔥🔥🔥

Count employees per department:

```java
Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Result:

```text
IT       → 3
HR       → 2
Finance  → 1
```

### Interview line

> `groupingBy()` creates the groups, while the downstream collector decides what to calculate inside each group.

---

# 7. `groupingBy()` + `mapping()` ⭐⭐⭐⭐⭐

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

Pipeline:

```text
Employee
   ↓
group by department
   ↓
map Employee → name
   ↓
collect List<String>
```

---

# 8. Grouping + `toSet()`

Department → unique employee names:

```java
Map<String, Set<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.toSet()
                )
        ));
```

---

# 9. Grouping + Salary Sum 🔥🔥🔥

Total salary per department:

```java
Map<String, Integer> salaryByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summingInt(Employee::getSalary)
        ));
```

Example:

```text
IT → 420000
HR → 220000
```

---

# 10. Grouping + Average Salary

```java
Map<String, Double> avgSalaryByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

---

# 11. Grouping + Max Salary ⭐⭐⭐⭐⭐

Highest salary per department:

```java
Map<String, Optional<Employee>> highestPaidByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

This returns `Optional<Employee>` for each group because a collector such as `maxBy()` represents the possibility of no value.

---

# 12. Grouping + Min Salary

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

# 13. Grouping + `summarizingInt()` 🔥

```java
Map<String, IntSummaryStatistics> salaryStats = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

For each department:

```java
stats.getCount();
stats.getSum();
stats.getMin();
stats.getMax();
stats.getAverage();
```

---

# 14. Grouping by Multiple Conditions

Example requirement:

> Group employees by department and then by designation.

```java
Map<String, Map<String, List<Employee>>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.groupingBy(Employee::getDesignation)
        ));
```

Result shape:

```text
Department
   ↓
Designation
   ↓
List<Employee>
```

---

# 15. Nested Grouping — Interview Level 🔥🔥🔥

```java
Map<String, Map<String, Long>> countByDepartmentAndRole = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.groupingBy(
                        Employee::getDesignation,
                        Collectors.counting()
                )
        ));
```

This is a strong 5-year experience Stream question.

---

# 16. Grouping With `LinkedHashMap`

The three-argument overload lets you choose the Map implementation:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                LinkedHashMap::new,
                Collectors.toList()
        ));
```

This is useful when you explicitly need insertion-order map behavior.

### Interview point

> The default `groupingBy()` does not promise a particular Map implementation; use a map factory when a specific implementation is required.

---

# 17. `groupingBy()` vs `groupingByConcurrent()` ⭐⭐⭐⭐⭐

Sequential/general grouping:

```java
Collectors.groupingBy(Employee::getDepartment)
```

Concurrent grouping for parallel processing:

```java
Collectors.groupingByConcurrent(Employee::getDepartment)
```

`groupingByConcurrent()` returns a concurrent map and is designed for concurrent reduction when its collector/source characteristics make that useful.

### Interview answer

> `groupingBy()` is the normal grouping collector. `groupingByConcurrent()` can be used for concurrent grouping and returns a `ConcurrentMap`, particularly when processing in parallel.

---

# 18. `partitioningBy()` Fundamentals ⭐⭐⭐⭐⭐

Partition employees based on salary:

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

---

# 19. `partitioningBy()` + Counting

```java
Map<Boolean, Long> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() > 100000,
                Collectors.counting()
        ));
```

Result:

```text
true  → high-paid count
false → lower/equal-paid count
```

---

# 20. `partitioningBy()` + Mapping

```java
Map<Boolean, List<String>> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() > 100000,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.toList()
                )
        ));
```

---

# 21. `partitioningBy()` + Summarizing

```java
Map<Boolean, IntSummaryStatistics> result = employees.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() > 100000,
                Collectors.summarizingInt(Employee::getSalary)
        ));
```

Now you can inspect count, sum, min, max and average for both partitions.

---

# 22. `groupingBy()` vs `partitioningBy()` 🔥🔥🔥

| Requirement | Use |
|---|---|
| Group by department | `groupingBy()` |
| Group by city | `groupingBy()` |
| Group by role | `groupingBy()` |
| Split high/low salary | `partitioningBy()` |
| Split active/inactive | `partitioningBy()` |
| Split pass/fail | `partitioningBy()` |
| Many categories | `groupingBy()` |
| Boolean condition | `partitioningBy()` |

### Golden rule

```text
Many possible keys → groupingBy()
Boolean split      → partitioningBy()
```

---

# 23. Important Difference — Boolean Grouping vs Partitioning

You could write:

```java
Collectors.groupingBy(e -> e.getSalary() > 100000)
```

but when the requirement is explicitly a boolean partition, prefer:

```java
Collectors.partitioningBy(e -> e.getSalary() > 100000)
```

`partitioningBy()` expresses the intent directly and is specifically designed for boolean predicates.

---

# 24. Find Highest-Paid Employee Per Department 🔥🔥🔥

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

This is one of the most common interview coding questions.

---

# 25. Find Highest Salary Per Department

If only the salary is needed:

```java
Map<String, Integer> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        ),
                        optional -> optional.map(Employee::getSalary).orElse(0)
                )
        ));
```

---

# 26. `collectingAndThen()` ⭐⭐⭐⭐⭐

`collectingAndThen()` performs a downstream collection and then transforms the result.

Example:

```java
Map<String, Integer> employeeCount = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.counting(),
                        Long::intValue
                )
        ));
```

Mental model:

```text
stream
 ↓
group
 ↓
collect
 ↓
post-process result
```

---

# 27. Top Employee Per Department ⭐⭐⭐⭐⭐

```java
Map<String, Employee> topEmployeeByDepartment = employees.stream()
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

This assumes every department group has at least one employee.

---

# 28. Grouping + Sorted Values 🔥

Department → employees sorted by salary descending:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.toList(),
                        list -> list.stream()
                                .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
                                .collect(Collectors.toList())
                )
        ));
```

An alternative is to sort the stream before grouping when global ordering characteristics are appropriate, but if the requirement is specifically ordering within each group, make that intent explicit and verify the desired Map/List ordering.

---

# 29. Grouping + Joining

Department → comma-separated employee names:

```java
Map<String, String> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.joining(", ")
                )
        ));
```

---

# 30. Grouping by Salary Range ⭐⭐⭐⭐⭐

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(e -> {
            if (e.getSalary() >= 150000) return "HIGH";
            if (e.getSalary() >= 100000) return "MEDIUM";
            return "LOW";
        }));
```

Output shape:

```text
HIGH   → [...]
MEDIUM → [...]
LOW    → [...]
```

This demonstrates that the classifier can contain business categorization logic.

---

# 31. Grouping by Normalized Key

Case-insensitive grouping:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                e -> e.getDepartment().toUpperCase(Locale.ROOT)
        ));
```

Useful when source data can contain inconsistent casing.

---

# 32. `toMap()` vs `groupingBy()` 🔥🔥🔥

### Unique key expected

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity()
        ));
```

### Multiple employees per key

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

### Interview line

> `toMap()` represents one value per key, while `groupingBy()` naturally represents a key with multiple values or a downstream aggregation.

---

# 33. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class GroupingPartitioningDemo {

    static class Employee {
        private final int id;
        private final String name;
        private final String department;
        private final String designation;
        private final int salary;

        Employee(int id, String name, String department,
                 String designation, int salary) {
            this.id = id;
            this.name = name;
            this.department = department;
            this.designation = designation;
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

        public String getDesignation() {
            return designation;
        }

        public int getSalary() {
            return salary;
        }

        @Override
        public String toString() {
            return id + " - " + name + " - " + department
                    + " - " + designation + " - " + salary;
        }
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", "Developer", 150000),
                new Employee(2, "Rahul", "IT", "Developer", 120000),
                new Employee(3, "Amit", "IT", "Lead", 180000),
                new Employee(4, "Priya", "HR", "Manager", 130000),
                new Employee(5, "Ravi", "HR", "Executive", 90000),
                new Employee(6, "Sneha", "Finance", "Analyst", 110000),
                new Employee(7, "Karan", "Finance", "Manager", 160000)
        );

        // 1. Basic grouping
        Map<String, List<Employee>> byDepartment = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment));

        // 2. Count by department
        Map<String, Long> countByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.counting()
                ));

        // 3. Department -> names
        Map<String, List<String>> namesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        )
                ));

        // 4. Department -> unique names
        Map<String, Set<String>> uniqueNamesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toSet()
                        )
                ));

        // 5. Total salary by department
        Map<String, Integer> salaryByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.summingInt(Employee::getSalary)
                ));

        // 6. Average salary by department
        Map<String, Double> avgSalaryByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.averagingInt(Employee::getSalary)
                ));

        // 7. Highest-paid employee per department
        Map<String, Optional<Employee>> highestPaidByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 8. Lowest-paid employee per department
        Map<String, Optional<Employee>> lowestPaidByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.minBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 9. Salary statistics per department
        Map<String, IntSummaryStatistics> salaryStats = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.summarizingInt(Employee::getSalary)
                ));

        // 10. Partition by salary
        Map<Boolean, List<Employee>> salaryPartition = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() > 100000
                ));

        // 11. Partition + count
        Map<Boolean, Long> salaryPartitionCount = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() > 100000,
                        Collectors.counting()
                ));

        // 12. Partition + names
        Map<Boolean, List<String>> salaryPartitionNames = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.getSalary() > 100000,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        )
                ));

        // 13. Department + designation nested grouping
        Map<String, Map<String, List<Employee>>> nestedGrouping = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.groupingBy(Employee::getDesignation)
                ));

        // 14. Department + designation counts
        Map<String, Map<String, Long>> nestedCounts = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.groupingBy(
                                Employee::getDesignation,
                                Collectors.counting()
                        )
                ));

        // 15. LinkedHashMap when explicit map type is required
        Map<String, List<Employee>> orderedGrouping = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        LinkedHashMap::new,
                        Collectors.toList()
                ));

        // 16. Department -> comma-separated names
        Map<String, String> joinedNames = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.joining(", ")
                        )
                ));

        // 17. Salary range grouping
        Map<String, List<Employee>> salaryRange = employees.stream()
                .collect(Collectors.groupingBy(e -> {
                    if (e.getSalary() >= 150000) return "HIGH";
                    if (e.getSalary() >= 100000) return "MEDIUM";
                    return "LOW";
                }));

        // 18. Highest salary value per department
        Map<String, Integer> highestSalaryByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.collectingAndThen(
                                Collectors.maxBy(
                                        Comparator.comparingInt(Employee::getSalary)
                                ),
                                optional -> optional.map(Employee::getSalary).orElse(0)
                        )
                ));

        // 19. Top employee per department
        Map<String, Employee> topEmployeeByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.collectingAndThen(
                                Collectors.maxBy(
                                        Comparator.comparingInt(Employee::getSalary)
                                ),
                                Optional::orElseThrow
                        )
                ));

        // 20. Group high-paid IT employees and return names
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

        System.out.println("By department: " + byDepartment);
        System.out.println("Count: " + countByDepartment);
        System.out.println("Names: " + namesByDepartment);
        System.out.println("Unique names: " + uniqueNamesByDepartment);
        System.out.println("Salary sum: " + salaryByDepartment);
        System.out.println("Average salary: " + avgSalaryByDepartment);
        System.out.println("Highest paid: " + highestPaidByDepartment);
        System.out.println("Lowest paid: " + lowestPaidByDepartment);
        System.out.println("Salary stats: " + salaryStats);
        System.out.println("Salary partition: " + salaryPartition);
        System.out.println("Partition count: " + salaryPartitionCount);
        System.out.println("Partition names: " + salaryPartitionNames);
        System.out.println("Nested grouping: " + nestedGrouping);
        System.out.println("Nested counts: " + nestedCounts);
        System.out.println("Ordered grouping: " + orderedGrouping);
        System.out.println("Joined names: " + joinedNames);
        System.out.println("Salary range: " + salaryRange);
        System.out.println("Highest salary: " + highestSalaryByDepartment);
        System.out.println("Top employee: " + topEmployeeByDepartment);
        System.out.println("High-paid IT: " + highPaidIT);
    }
}
```

---

# 34. Output Prediction 🔥

What is the result?

```java
Map<String, Long> result = Arrays.asList(
        "IT", "HR", "IT", "Finance", "HR", "IT"
).stream().collect(Collectors.groupingBy(
        Function.identity(),
        Collectors.counting()
));
```

Expected counts:

```text
IT      → 3
HR      → 2
Finance → 1
```

---

# 35. Output Prediction — Partitioning ⭐⭐⭐⭐⭐

```java
Map<Boolean, List<Integer>> result = Arrays.asList(1, 2, 3, 4, 5)
        .stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));
```

Result:

```text
true  → [2, 4]
false → [1, 3, 5]
```

---

# 36. Interview Trap — `groupingBy()` With Duplicate Keys

This is perfectly valid:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Multiple employees can belong to the same department because the value is a List.

This is different from:

```java
Collectors.toMap(Employee::getDepartment, Function.identity())
```

which requires a merge strategy when duplicate department keys exist.

---

# 37. Interview Trap — `partitioningBy()` Is Boolean

Do not think of `partitioningBy()` as a general grouping collector.

```java
partitioningBy(predicate)
```

is specifically based on a boolean predicate:

```text
true / false
```

If you need:

```text
IT / HR / Finance / Sales
```

use `groupingBy()`.

---

# 38. Interview Scenario — Highest Salary Per Department 🔥🔥🔥

**Question:** Find the highest-paid employee from every department.

**Answer:**

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

**Explain:**

```text
1. Stream employees
2. Create department groups
3. Apply maxBy inside each group
4. Compare salary
5. Return department → highest-paid employee
```

---

# 39. Interview Scenario — Total Salary Per Department

```java
Map<String, Integer> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summingInt(Employee::getSalary)
        ));
```

### Interview line

> I use a downstream collector when I need an aggregation per group rather than the complete list of objects.

---

# 40. Interview Scenario — Active vs Inactive

```java
Map<Boolean, List<Employee>> result = employees.stream()
        .collect(Collectors.partitioningBy(Employee::isActive));
```

This is cleaner than manually creating two lists.

---

# 41. Interview Scenario — Pass / Fail Students

```java
Map<Boolean, List<Student>> result = students.stream()
        .collect(Collectors.partitioningBy(
                student -> student.getMarks() >= 40
        ));
```

Mental model:

```text
true  → PASS
false → FAIL
```

---

# 42. Interview Scenario — Group Orders by Status

```java
Map<String, List<Order>> result = orders.stream()
        .collect(Collectors.groupingBy(Order::getStatus));
```

Then count:

```java
Map<String, Long> countByStatus = orders.stream()
        .collect(Collectors.groupingBy(
                Order::getStatus,
                Collectors.counting()
        ));
```

---

# 43. Interview Scenario — Group by Department, Then Find Average Salary

```java
Map<String, Double> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

---

# 44. Interview Scenario — Group by Department and Return Names

```java
Map<String, List<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.toList()
                )
        ));
```

---

# 45. Interview Scenario — Two-Level Grouping

```java
Map<String, Map<String, List<Employee>>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.groupingBy(Employee::getDesignation)
        ));
```

This pattern is useful when the result needs a hierarchy.

---

# 46. 25 Interview Questions 🔥🔥🔥

1. What does `groupingBy()` do?
2. What is a classifier in `groupingBy()`?
3. What is a downstream collector?
4. How do you group employees by department?
5. How do you count employees per department?
6. How do you calculate total salary per department?
7. How do you calculate average salary per department?
8. How do you find highest-paid employee per department?
9. Why does `maxBy()` return Optional?
10. How do you return department → employee names?
11. How do you group by two fields?
12. How do you preserve a specific Map implementation?
13. What is `groupingByConcurrent()`?
14. Difference between `groupingBy()` and `groupingByConcurrent()`?
15. What does `partitioningBy()` do?
16. When should you use `partitioningBy()`?
17. Difference between `groupingBy()` and `partitioningBy()`?
18. Can `partitioningBy()` have a downstream collector?
19. How do you count true/false partitions?
20. How do you partition employees into high/low salary?
21. What is `collectingAndThen()`?
22. How do you get the highest salary value per department?
23. Difference between `toMap()` and `groupingBy()`?
24. How would you group orders by status and count them?
25. Explain a nested `groupingBy()` pipeline in 2 minutes.

---

# 47. Coding Challenges 🔥🔥🔥

### Challenge 1
Group employees by department.

### Challenge 2
Count employees by department.

### Challenge 3
Calculate total salary by department.

### Challenge 4
Calculate average salary by department.

### Challenge 5
Find highest-paid employee per department.

### Challenge 6
Find lowest-paid employee per department.

### Challenge 7
Return department → employee names.

### Challenge 8
Return department → unique employee names.

### Challenge 9
Group employees by department and designation.

### Challenge 10
Count employees by department and designation.

### Challenge 11
Partition employees by salary > 10 LPA.

### Challenge 12
Partition students into pass/fail.

### Challenge 13
Partition employees into active/inactive.

### Challenge 14
Find salary statistics per department.

### Challenge 15
Group orders by status and count them.

### Challenge 16 ⭐⭐⭐⭐⭐
Find highest salary per department without returning `Optional` in the final Map.

### Challenge 17 ⭐⭐⭐⭐⭐
Return department → employee names sorted alphabetically.

### Challenge 18 ⭐⭐⭐⭐⭐
Return department → employees sorted by salary descending.

### Challenge 19
Find total salary of employees in each designation.

### Challenge 20
Find average salary of each designation within each department.

### Challenge 21 — 5-Year Interview Level 🔥
Create a nested result:

```text
Department → Designation → Employee Count
```

### Challenge 22 — 5-Year Interview Level 🔥
Create:

```text
Department → Highest Paid Employee
```

and explain how you handle empty groups.

### Challenge 23
Create:

```text
Department → Total Salary
```

then identify the department with the highest total salary.

### Challenge 24
Partition employees by salary and calculate average salary in each partition.

### Challenge 25 — Production Scenario 🔥🔥🔥
Given orders containing `status`, `customerId`, `amount` and `createdDate`, produce status-wise order count, total amount and average amount using downstream collectors.

---

# 48. Common Mistakes ❌

### ❌ Mistake 1
Using `partitioningBy()` for multiple categories.

```text
IT / HR / Finance
```

Use `groupingBy()`.

### ❌ Mistake 2
Using `toMap()` when duplicate keys are expected.

Use `groupingBy()` or provide a merge function.

### ❌ Mistake 3
Forgetting the downstream collector.

```java
groupingBy(Employee::getDepartment, counting())
```
means count per group, not list per group.

### ❌ Mistake 4
Ignoring `Optional` from `maxBy()` / `minBy()`.

### ❌ Mistake 5
Assuming the default Map implementation/order from `groupingBy()`.

Use a map factory when a specific Map implementation is required.

### ❌ Mistake 6
Using nested grouping when a simple composite key or domain object would make the model clearer.

Choose the result shape based on the business requirement.

---

# 49. Final Revision Sheet 🧠

```text
groupingBy()
────────────────────────────
classifier → group key
result      → Map<K, ...>
```

```text
groupingBy() + counting()
────────────────────────────
K → count
```

```text
groupingBy() + mapping()
────────────────────────────
K → mapped values
```

```text
groupingBy() + summingInt()
────────────────────────────
K → total
```

```text
groupingBy() + averagingInt()
────────────────────────────
K → average
```

```text
groupingBy() + maxBy()
────────────────────────────
K → max element
```

```text
partitioningBy(predicate)
────────────────────────────
true → group 1
false → group 2
```

### Golden Rules

```text
Many categories?        → groupingBy()
Boolean condition?      → partitioningBy()
Count per group?        → counting()
Sum per group?          → summingInt()
Average per group?      → averagingInt()
Max per group?          → maxBy()
Names per group?        → mapping()
Post-process result?    → collectingAndThen()
Concurrent grouping?    → groupingByConcurrent()
```

---

# 50. 2-Minute Interview Script 🎤

> “`groupingBy()` is a Collector used when I need to divide stream elements into groups based on a classifier. The default result is effectively a key-to-list mapping, but I can provide a downstream collector such as `counting()`, `summingInt()`, `averagingInt()`, `mapping()` or `maxBy()` to calculate an aggregation per group. For example, I can calculate total salary per department using `groupingBy(Employee::getDepartment, summingInt(Employee::getSalary))`. `partitioningBy()` is specifically designed for a boolean predicate and creates true and false partitions, such as high-paid versus low-paid employees. If I have many categories like IT, HR and Finance, I use `groupingBy()`; if I have a binary condition like pass/fail or active/inactive, I use `partitioningBy()`. For concurrent parallel grouping, `groupingByConcurrent()` can be considered when appropriate.”

---

# 🧪 Complete Practice Code

[GitHub — 9.15 groupingBy / partitioningBy Deep Dive Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/15-Collectors-groupingBy-partitioningBy-Deep-Dive)

---

## Navigation

[← 9.14 — Stream reduce / collect Fundamentals](../14-Stream-reduce-collect-Fundamentals/README.md)

**Current → 9.15 — `Collectors.groupingBy()` / `partitioningBy()` Deep Dive → ✅ Completed**

**Next → 9.16 — `Collectors.toMap()` / `toSet()` / `toList()` Deep Dive**