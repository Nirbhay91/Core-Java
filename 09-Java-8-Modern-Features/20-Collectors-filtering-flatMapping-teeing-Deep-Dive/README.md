# 9.20 — `Collectors.filtering()` / `flatMapping()` / `teeing()` Deep Dive

## 🎯 Interview Goal

Master three powerful collectors used mainly as **downstream collectors**:

```text
filtering()  → filter elements inside a collector/group
flatMapping() → flatten nested values inside a collector/group
teeing()     → run two collectors and merge their results
```

> **Version note:** `filtering()` and `flatMapping()` were introduced in Java 9. `teeing()` was introduced in Java 12. If your interview/project is Java 8-only, do not claim these APIs are available in Java 8.

---

# 1. `Collectors.filtering()` Fundamentals ⭐⭐⭐⭐⭐

`filtering()` allows filtering to happen **inside a downstream collector**.

Basic example:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.toList()
                )
        ));
```

Mental model:

```text
Employees
   ↓
groupingBy(department)
   ↓
filtering(salary >= 100000)
   ↓
toList()
   ↓
Department → filtered employees
```

---

# 2. `stream.filter()` vs `Collectors.filtering()` 🔥🔥🔥

### `stream.filter()`

```java
Map<String, List<Employee>> result = employees.stream()
        .filter(e -> e.getSalary() >= 100000)
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Departments with **no qualifying employee can disappear**.

### `Collectors.filtering()`

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.toList()
                )
        ));
```

The existing groups are retained and their downstream lists can be empty.

### Interview line

> `stream.filter()` removes elements before grouping, while `Collectors.filtering()` applies the predicate inside each downstream group.

---

# 3. `filtering()` + `counting()` 🔥🔥🔥

Count high earners per department while preserving departments:

```java
Map<String, Long> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.counting()
                )
        ));
```

---

# 4. `filtering()` + `summingInt()`

Total salary of employees earning at least 100000 per department:

```java
Map<String, Integer> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.summingInt(Employee::getSalary)
                )
        ));
```

---

# 5. `filtering()` + `averagingInt()`

```java
Map<String, Double> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.averagingInt(Employee::getSalary)
                )
        ));
```

Remember that an empty downstream group produces the collector's empty-result behavior.

---

# 6. `filtering()` + `summarizingInt()` 🔥🔥🔥

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

This is useful when you need filtered count, sum, min, average and max per group.

---

# 7. `filtering()` + `maxBy()`

Highest-paid employee earning at least 100000 in each department:

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                )
        ));
```

---

# 8. `filtering()` + `mapping()`

Get names of high earners per department:

```java
Map<String, List<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        )
                )
        ));
```

This combines two downstream transformations:

```text
filtering → mapping → toList
```

---

# 9. `flatMapping()` Fundamentals ⭐⭐⭐⭐⭐

`flatMapping()` is the downstream-collector equivalent of the idea behind `flatMap()`.

Suppose each employee has multiple skills:

```java
class Employee {
    private String name;
    private String department;
    private List<String> skills;
}
```

Collect all skills per department:

```java
Map<String, Set<String>> skillsByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.flatMapping(
                        e -> e.getSkills().stream(),
                        Collectors.toSet()
                )
        ));
```

Mental model:

```text
Employee
   ↓
groupingBy(department)
   ↓
flatMapping(employee → skills.stream())
   ↓
toSet()
   ↓
Department → all unique skills
```

---

# 10. `flatMapping()` vs `mapping()` 🔥🔥🔥

### `mapping()`

One input → one mapped value:

```java
Collectors.mapping(
        Employee::getName,
        Collectors.toList()
)
```

Conceptually:

```text
Employee → Name
```

### `flatMapping()`

One input → zero/many values:

```java
Collectors.flatMapping(
        e -> e.getSkills().stream(),
        Collectors.toSet()
)
```

Conceptually:

```text
Employee → Stream<Skill> → flattened skills
```

### Interview line

> `mapping()` transforms each element into one downstream value, while `flatMapping()` lets each element contribute multiple values and flattens those values into the downstream collector.

---

# 11. `flatMapping()` + `joining()` 🔥

Collect all unique skills as a comma-separated string per department:

```java
Map<String, String> skillsByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.flatMapping(
                        e -> e.getSkills().stream(),
                        Collectors.collectingAndThen(
                                Collectors.toSet(),
                                set -> set.stream()
                                        .sorted()
                                        .collect(Collectors.joining(", "))
                        )
                )
        ));
```

---

# 12. `flatMapping()` + `counting()`

Count skill occurrences per department:

```java
Map<String, Long> skillOccurrencesByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.flatMapping(
                        e -> e.getSkills().stream(),
                        Collectors.counting()
                )
        ));
```

For **unique skill count**, use `toSet()` first:

```java
Map<String, Integer> uniqueSkillCountByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.flatMapping(
                                e -> e.getSkills().stream(),
                                Collectors.toSet()
                        ),
                        Set::size
                )
        ));
```

---

# 13. Null-Safe `flatMapping()` ⚠️

If `getSkills()` can return `null`, avoid:

```java
Collectors.flatMapping(
        e -> e.getSkills().stream(),
        Collectors.toSet()
)
```

because it can throw `NullPointerException`.

Safer approach:

```java
Collectors.flatMapping(
        e -> e.getSkills() == null
                ? Stream.empty()
                : e.getSkills().stream(),
        Collectors.toSet()
)
```

---

# 14. `flatMapping()` With Nested Objects 🔥

Suppose:

```java
class Employee {
    private List<Project> projects;
}
```

Then:

```java
Map<String, Set<Project>> projectsByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.flatMapping(
                        e -> e.getProjects().stream(),
                        Collectors.toSet()
                )
        ));
```

This is useful for employee → projects, order → products, customer → orders, etc.

---

# 15. `teeing()` Fundamentals ⭐⭐⭐⭐⭐

`teeing()` runs **two collectors over the same stream** and combines their results using a merger function.

Example: calculate total and average salary together:

```java
Map<String, Object> result = employees.stream()
        .collect(Collectors.teeing(
                Collectors.summingInt(Employee::getSalary),
                Collectors.averagingInt(Employee::getSalary),
                (sum, average) -> Map.of(
                        "sum", sum,
                        "average", average
                )
        ));
```

Mental model:

```text
                 ┌── summingInt() ──┐
Employees ───────┤                   ├── merger ──→ result
                 └── averagingInt() ─┘
```

---

# 16. `teeing()` With `minBy()` + `maxBy()` 🔥🔥🔥

Find both minimum and maximum salary employee in one collector:

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.teeing(
                Collectors.minBy(Comparator.comparingInt(Employee::getSalary)),
                Collectors.maxBy(Comparator.comparingInt(Employee::getSalary)),
                (min, max) -> {
                    Map<String, Employee> map = new HashMap<>();
                    min.ifPresent(e -> map.put("min", e));
                    max.ifPresent(e -> map.put("max", e));
                    return map;
                }
        ));
```

---

# 17. Prefer a Typed Result Over `Map<String, Object>` 🔥🔥🔥

For production-quality code, create a result type:

```java
record SalarySummary(
        int total,
        double average
) {}
```

Then:

```java
SalarySummary summary = employees.stream()
        .collect(Collectors.teeing(
                Collectors.summingInt(Employee::getSalary),
                Collectors.averagingInt(Employee::getSalary),
                SalarySummary::new
        ));
```

For Java 16+ records are available. On older Java versions, use a normal POJO.

---

# 18. `teeing()` + `groupingBy()` 🔥🔥🔥

Department → total and average salary:

```java
Map<String, SalarySummary> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.teeing(
                        Collectors.summingInt(Employee::getSalary),
                        Collectors.averagingInt(Employee::getSalary),
                        SalarySummary::new
                )
        ));
```

This is an advanced downstream collector pattern.

---

# 19. `teeing()` + `counting()` + `summarizingInt()`

Create a dashboard containing count and salary statistics:

```java
record DepartmentSummary(
        long count,
        IntSummaryStatistics salaryStats
) {}

Map<String, DepartmentSummary> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.teeing(
                        Collectors.counting(),
                        Collectors.summarizingInt(Employee::getSalary),
                        DepartmentSummary::new
                )
        ));
```

Note: `summarizingInt()` already contains count, so this example intentionally demonstrates combining two independent downstream collectors.

---

# 20. `teeing()` + `filtering()` 🔥

Calculate count and average only for high earners:

```java
record HighEarnerSummary(long count, double average) {}

HighEarnerSummary result = employees.stream()
        .collect(Collectors.teeing(
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.counting()
                ),
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.averagingInt(Employee::getSalary)
                ),
                HighEarnerSummary::new
        ));
```

---

# 21. `teeing()` Is Not Just “Run Two Streams”

Conceptually:

```text
teeing(collector1, collector2, merger)
```

means:

```text
same input stream
      ↓
 ┌────┴────┐
 ↓         ↓
C1        C2
 ↓         ↓
R1        R2
 └────┬────┘
      ↓
   merger
      ↓
   final R
```

It is a **collector composition** mechanism.

---

# 22. `teeing()` vs Multiple Terminal Operations 🔥

Without `teeing()`:

```java
int sum = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();

double average = employees.stream()
        .mapToInt(Employee::getSalary)
        .average()
        .orElse(0.0);
```

This traverses the source separately.

With `teeing()`:

```java
SalarySummary summary = employees.stream()
        .collect(Collectors.teeing(
                Collectors.summingInt(Employee::getSalary),
                Collectors.averagingInt(Employee::getSalary),
                SalarySummary::new
        ));
```

The collector coordinates both downstream collectors over the same collection operation.

---

# 23. `filtering()` + `flatMapping()` 🔥🔥🔥

Get skills only from high earners per department:

```java
Map<String, Set<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.flatMapping(
                                e -> e.getSkills().stream(),
                                Collectors.toSet()
                        )
                )
        ));
```

Mental model:

```text
Employee
   ↓
filter salary
   ↓
extract skills
   ↓
flatten
   ↓
unique Set
```

---

# 24. `filtering()` + `flatMapping()` + `joining()` 🔥🔥🔥

```java
Map<String, String> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.flatMapping(
                                e -> e.getSkills().stream(),
                                Collectors.collectingAndThen(
                                        Collectors.toSet(),
                                        set -> set.stream()
                                                .sorted()
                                                .collect(Collectors.joining(", "))
                                )
                        )
                )
        ));
```

This is a strong advanced interview problem.

---

# 25. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class FilteringFlatMappingTeeingDemo {

    static class Employee {
        private final int id;
        private final String name;
        private final String department;
        private final int salary;
        private final List<String> skills;

        Employee(int id, String name, String department,
                 int salary, List<String> skills) {
            this.id = id;
            this.name = name;
            this.department = department;
            this.salary = salary;
            this.skills = skills;
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

        public List<String> getSkills() {
            return skills;
        }

        @Override
        public String toString() {
            return id + " - " + name + " - " + department
                    + " - " + salary + " - " + skills;
        }
    }

    static class SalarySummary {
        private final int total;
        private final double average;

        SalarySummary(int total, double average) {
            this.total = total;
            this.average = average;
        }

        @Override
        public String toString() {
            return "SalarySummary{total=" + total
                    + ", average=" + average + "}";
        }
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000,
                        Arrays.asList("Java", "Spring", "SQL")),
                new Employee(2, "Rahul", "IT", 120000,
                        Arrays.asList("Java", "Docker", "SQL")),
                new Employee(3, "Amit", "IT", 180000,
                        Arrays.asList("Java", "AWS", "Spring")),
                new Employee(4, "Priya", "HR", 130000,
                        Arrays.asList("Excel", "Recruitment")),
                new Employee(5, "Ravi", "HR", 90000,
                        Arrays.asList("Recruitment", "Communication")),
                new Employee(6, "Sneha", "Finance", 110000,
                        Arrays.asList("Excel", "SQL"))
        );

        // 1. filtering(): high earners per department
        Map<String, List<Employee>> highEarners = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.toList()
                        )
                ));

        // 2. filtering() + counting(): high-earner count per department
        Map<String, Long> highEarnerCount = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.counting()
                        )
                ));

        // 3. filtering() + summingInt(): high-earner salary total per department
        Map<String, Integer> highEarnerSalaryTotal = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.summingInt(Employee::getSalary)
                        )
                ));

        // 4. filtering() + mapping(): names of high earners per department
        Map<String, List<String>> highEarnerNames = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.mapping(
                                        Employee::getName,
                                        Collectors.toList()
                                )
                        )
                ));

        // 5. filtering() + maxBy(): highest high-earner per department
        Map<String, Optional<Employee>> highestHighEarner = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.maxBy(
                                        Comparator.comparingInt(Employee::getSalary)
                                )
                        )
                ));

        // 6. flatMapping(): all unique skills per department
        Map<String, Set<String>> skillsByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.flatMapping(
                                e -> e.getSkills().stream(),
                                Collectors.toSet()
                        )
                ));

        // 7. filtering() + flatMapping(): skills of high earners per department
        Map<String, Set<String>> highEarnerSkillsByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.flatMapping(
                                        e -> e.getSkills().stream(),
                                        Collectors.toSet()
                                )
                        )
                ));

        // 8. teeing(): total + average salary
        SalarySummary overallSalarySummary = employees.stream()
                .collect(Collectors.teeing(
                        Collectors.summingInt(Employee::getSalary),
                        Collectors.averagingInt(Employee::getSalary),
                        SalarySummary::new
                ));

        // 9. teeing() + groupingBy(): department salary summary
        Map<String, SalarySummary> salarySummaryByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.teeing(
                                Collectors.summingInt(Employee::getSalary),
                                Collectors.averagingInt(Employee::getSalary),
                                SalarySummary::new
                        )
                ));

        // 10. teeing(): minimum + maximum employee
        Map<String, Employee> minMax = employees.stream()
                .collect(Collectors.teeing(
                        Collectors.minBy(Comparator.comparingInt(Employee::getSalary)),
                        Collectors.maxBy(Comparator.comparingInt(Employee::getSalary)),
                        (min, max) -> {
                            Map<String, Employee> result = new HashMap<>();
                            min.ifPresent(e -> result.put("min", e));
                            max.ifPresent(e -> result.put("max", e));
                            return result;
                        }
                ));

        // 11. teeing() + filtering(): high-earner count + average
        Map<String, Object> highEarnerSummary = employees.stream()
                .collect(Collectors.teeing(
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.counting()
                        ),
                        Collectors.filtering(
                                e -> e.getSalary() >= 100000,
                                Collectors.averagingInt(Employee::getSalary)
                        ),
                        (count, average) -> {
                            Map<String, Object> result = new HashMap<>();
                            result.put("count", count);
                            result.put("average", average);
                            return result;
                        }
                ));

        // 12. Empty source with teeing()
        SalarySummary emptySummary = Collections.<Employee>emptyList()
                .stream()
                .collect(Collectors.teeing(
                        Collectors.summingInt(Employee::getSalary),
                        Collectors.averagingInt(Employee::getSalary),
                        SalarySummary::new
                ));

        // 13. Null-safe flatMapping example
        List<Employee> employeesWithNullableSkills = Arrays.asList(
                new Employee(7, "Test", "IT", 100000, null),
                new Employee(8, "Demo", "IT", 110000,
                        Arrays.asList("Java", "SQL"))
        );

        Map<String, Set<String>> nullSafeSkills = employeesWithNullableSkills.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.flatMapping(
                                e -> e.getSkills() == null
                                        ? Stream.empty()
                                        : e.getSkills().stream(),
                                Collectors.toSet()
                        )
                ));

        System.out.println("High earners: " + highEarners);
        System.out.println("High-earner count: " + highEarnerCount);
        System.out.println("High-earner salary total: " + highEarnerSalaryTotal);
        System.out.println("High-earner names: " + highEarnerNames);
        System.out.println("Highest high-earner: " + highestHighEarner);
        System.out.println("Skills by department: " + skillsByDepartment);
        System.out.println("High-earner skills: " + highEarnerSkillsByDepartment);
        System.out.println("Overall salary summary: " + overallSalarySummary);
        System.out.println("Department salary summary: " + salarySummaryByDepartment);
        System.out.println("Min/Max: " + minMax);
        System.out.println("High-earner summary: " + highEarnerSummary);
        System.out.println("Empty summary: " + emptySummary);
        System.out.println("Null-safe skills: " + nullSafeSkills);
    }
}
```

---

# 26. Interview Scenario — Preserve Departments While Filtering 🔥🔥🔥

**Question:** Show departments with only employees earning >= 100000, but do not lose departments that have no qualifying employee.

### Answer

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.toList()
                )
        ));
```

### Why not just `stream.filter()`?

```java
employees.stream()
        .filter(e -> e.getSalary() >= 100000)
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

This filters before grouping, so a department with zero matching employees will not appear in the resulting map.

---

# 27. Interview Scenario — All Skills Per Department 🔥🔥🔥

```java
Map<String, Set<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.flatMapping(
                        e -> e.getSkills().stream(),
                        Collectors.toSet()
                )
        ));
```

### Why `flatMapping()`?

Each employee has a collection of skills:

```text
Employee 1 → [Java, SQL]
Employee 2 → [Java, Docker]
```

We want:

```text
[Java, SQL, Java, Docker]
```

then `toSet()` gives:

```text
[Java, SQL, Docker]
```

---

# 28. Interview Scenario — Department Salary Summary With `teeing()` 🔥🔥🔥

**Question:** Return total and average salary for every department.

```java
Map<String, SalarySummary> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.teeing(
                        Collectors.summingInt(Employee::getSalary),
                        Collectors.averagingInt(Employee::getSalary),
                        SalarySummary::new
                )
        ));
```

Mental model:

```text
Department group
      ↓
 ┌────┴─────┐
 ↓          ↓
sum       average
 └────┬─────┘
      ↓
SalarySummary
```

---

# 29. Interview Scenario — `teeing()` for Min + Max

```java
record MinMax(Optional<Employee> min, Optional<Employee> max) {}

MinMax result = employees.stream()
        .collect(Collectors.teeing(
                Collectors.minBy(Comparator.comparingInt(Employee::getSalary)),
                Collectors.maxBy(Comparator.comparingInt(Employee::getSalary)),
                MinMax::new
        ));
```

For Java versions without records, use a normal class.

---

# 30. Interview Scenario — High Earners and Skills 🔥🔥🔥

**Question:** For each department, return unique skills belonging only to employees earning >= 100000.

```java
Map<String, Set<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.filtering(
                        e -> e.getSalary() >= 100000,
                        Collectors.flatMapping(
                                e -> e.getSkills().stream(),
                                Collectors.toSet()
                        )
                )
        ));
```

This combines three concepts:

```text
filtering()
    ↓
flatMapping()
    ↓
toSet()
```

---

# 31. 25 Interview Questions 🎯

1. What is `Collectors.filtering()`?
2. When was `filtering()` introduced?
3. Difference between `stream.filter()` and downstream `filtering()`?
4. Why can `filtering()` preserve empty groups?
5. How do you combine `filtering()` with `groupingBy()`?
6. What is `Collectors.flatMapping()`?
7. When was `flatMapping()` introduced?
8. Difference between `mapping()` and `flatMapping()`?
9. How do you collect all employee skills per department?
10. How do you combine `flatMapping()` with `toSet()`?
11. How do you combine `flatMapping()` with `joining()`?
12. How do you make a `flatMapping()` pipeline null-safe?
13. What is `Collectors.teeing()`?
14. When was `teeing()` introduced?
15. How does `teeing()` work internally at a high level?
16. Can `teeing()` combine `summingInt()` and `averagingInt()`?
17. How do you use `teeing()` with `groupingBy()`?
18. How do you use `teeing()` with `minBy()` and `maxBy()`?
19. Why use a typed result object instead of `Map<String, Object>`?
20. Does `teeing()` mean two independent terminal operations?
21. How do `filtering()` and `flatMapping()` compose?
22. How do you calculate high-earner count and average together?
23. Explain `filtering + flatMapping + toSet`.
24. Explain `groupingBy + teeing` in a 5-year interview.
25. Which of these APIs are unavailable in Java 8?

---

# 32. Coding Challenges 💻

### Challenge 1
Use `filtering()` to retain employees earning >= 100000 inside each department.

### Challenge 2
Count high earners per department using `filtering()` + `counting()`.

### Challenge 3
Calculate high-earner salary total per department.

### Challenge 4
Find highest high-earner per department.

### Challenge 5
Collect names of high earners per department.

### Challenge 6
Collect all unique employee skills per department using `flatMapping()`.

### Challenge 7
Collect unique skills only from high earners.

### Challenge 8
Count all skill occurrences per department.

### Challenge 9
Return sorted comma-separated skills per department.

### Challenge 10
Make `flatMapping()` null-safe when an employee has no skills.

### Challenge 11 ⭐⭐⭐⭐⭐
Use `teeing()` to calculate total and average salary together.

### Challenge 12
Use `teeing()` to calculate min and max salary employee together.

### Challenge 13
Use `groupingBy()` + `teeing()` for department salary summaries.

### Challenge 14
Use `teeing()` to calculate count and average of high earners.

### Challenge 15
Use `filtering()` + `flatMapping()` + `toSet()` for high-earner skills.

### Challenge 16
Use `filtering()` + `flatMapping()` + `joining()` for a department skill report.

### Challenge 17
Create a typed `SalarySummary` result instead of `Map<String,Object>`.

### Challenge 18
Create a `MinMax` result containing two `Optional<Employee>` values.

### Challenge 19
Combine `filtering()` with `summarizingInt()`.

### Challenge 20
Combine `flatMapping()` with `mapping()` through a downstream pipeline.

### Challenge 21
Preserve all departments while calculating high-earner average salary.

### Challenge 22
Return all unique projects per department using `flatMapping()`.

### Challenge 23
Return unique technologies used by high-paid employees.

### Challenge 24
Compare `stream.filter()` and downstream `filtering()` using a department with zero matches.

### Challenge 25 — 5-Year Interview Level 🔥🔥🔥
Design a department analytics collector that returns:

```text
Employee count
High-earner count
Total salary
Average salary
Highest-paid employee
All unique skills
```

Use appropriate downstream collectors and explain the trade-offs.

---

# 33. Common Mistakes ❌

### ❌ Mistake 1 — Confusing Java versions

```text
filtering()   → Java 9+
flatMapping() → Java 9+
teeing()      → Java 12+
```

These are not Java 8 collectors.

### ❌ Mistake 2 — Assuming `filtering()` is identical to `stream.filter()`

Their position in the pipeline differs.

```text
stream.filter()
→ before grouping/collection

filtering()
→ inside downstream collector
```

### ❌ Mistake 3 — Using `mapping()` when one element produces many values

Use `flatMapping()` when the mapping produces a stream/collection of values that need flattening.

### ❌ Mistake 4 — Forgetting null handling in nested collections

```java
e.getSkills().stream()
```

can fail if `getSkills()` returns `null`.

### ❌ Mistake 5 — Using `Map<String,Object>` in production unnecessarily

Prefer a typed DTO/class/record for a multi-value result.

### ❌ Mistake 6 — Claiming `teeing()` performs arbitrary parallelism

`teeing()` is a collector composition mechanism. It should not be described as a general-purpose parallel execution API.

### ❌ Mistake 7 — Ignoring empty downstream results

`filtering()` can create empty groups, so understand how the downstream collector represents an empty result.

---

# 34. Final Revision Sheet 🧠

```text
filtering()
────────────────────────
Filter inside downstream collector
Java 9+
```

```text
flatMapping()
────────────────────────
Flatten nested values inside downstream collector
Java 9+
```

```text
teeing()
────────────────────────
Two downstream collectors + merger
Java 12+
```

### Golden Rules

```text
Need filter before grouping       → stream.filter()
Need filter inside each group     → filtering()
One input → one downstream value  → mapping()
One input → many values           → flatMapping()
Need two aggregations together    → teeing()
```

### Most important patterns

```java
Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.filtering(
                predicate,
                Collectors.toList()
        )
)
```

```java
Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.flatMapping(
                e -> e.getSkills().stream(),
                Collectors.toSet()
        )
)
```

```java
Collectors.teeing(
        collector1,
        collector2,
        merger
)
```

---

# 35. 2-Minute Interview Script 🎤

> “`Collectors.filtering()`, `flatMapping()` and `teeing()` are advanced collector APIs, and an important version point is that filtering and flatMapping were introduced in Java 9, while teeing was introduced in Java 12. `filtering()` applies a predicate inside a downstream collector, which is useful when I want to preserve the grouping structure even when a group has no matching elements. `flatMapping()` is useful when each input element contributes multiple values, such as an employee having multiple skills, and I want those values flattened into a downstream collection. `teeing()` combines two collectors operating over the same collection operation and merges their results, for example calculating total and average salary into a typed summary object. In an interview, I would also clearly distinguish these APIs from Java 8 equivalents because they are not available in Java 8.”

---

# 🧪 Complete Practice Code

[GitHub — 9.20 `filtering()` / `flatMapping()` / `teeing()` Deep Dive Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/20-Collectors-filtering-flatMapping-teeing-Deep-Dive)

---

## Navigation

[← 9.19 — `maxBy()` / `minBy()` / `counting()` Deep Dive](../19-Collectors-maxBy-minBy-counting-Deep-Dive/README.md)

**Current → 9.20 — `filtering()` / `flatMapping()` / `teeing()` → ✅ Completed**

**Next → 9.21 — Advanced Stream Performance & Parallel Streams**