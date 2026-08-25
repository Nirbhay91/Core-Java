# 9.17 — `Collectors.joining()` / `mapping()` / `collectingAndThen()` Deep Dive

## 🎯 Interview Goal

Master three powerful downstream collectors:

```text
joining()            → combine Strings
mapping()            → transform before downstream collection
collectingAndThen()  → collect, then post-process the result
```

These are especially important when combined with `groupingBy()`.

---

# 1. `Collectors.joining()` Fundamentals ⭐⭐⭐⭐⭐

Convert names into one String:

```java
String names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining());
```

With delimiter:

```java
String names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining(", "));
```

Output conceptually:

```text
Nirbhay, Rahul, Amit, Priya
```

---

# 2. `joining()` With Prefix and Suffix 🔥

```java
String result = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining(", ", "[", "]"));
```

Output:

```text
[Nirbhay, Rahul, Amit, Priya]
```

Signature:

```java
joining()
joining(delimiter)
joining(delimiter, prefix, suffix)
```

---

# 3. `joining()` Is a String Collector

`joining()` works with character sequences.

If the stream contains objects, map them to the required String first:

```java
String result = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining(", "));
```

For custom formatting:

```java
String result = employees.stream()
        .map(e -> e.getName() + "(" + e.getSalary() + ")")
        .collect(Collectors.joining(", "));
```

---

# 4. `mapping()` Fundamentals ⭐⭐⭐⭐⭐

`mapping()` is mainly useful as a **downstream collector**.

Example: department → employee names:

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

Mental model:

```text
Employee
   ↓
group by department
   ↓
map Employee → name
   ↓
collect names into List
```

---

# 5. `mapping()` + `toSet()`

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

# 6. `mapping()` + `joining()` 🔥🔥🔥

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

This is a very common interview pattern.

---

# 7. `mapping()` + `counting()`

Department → number of employees:

```java
Map<String, Long> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getName,
                        Collectors.counting()
                )
        ));
```

The mapping is unnecessary if every Employee should simply be counted, but it demonstrates how downstream collectors compose.

Simpler version:

```java
Map<String, Long> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

---

# 8. `mapping()` + `averagingInt()`

Department → average salary:

```java
Map<String, Double> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getSalary,
                        Collectors.averagingInt(Integer::intValue)
                )
        ));
```

Direct version is cleaner:

```java
Map<String, Double> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

---

# 9. `mapping()` + `summingInt()`

```java
Map<String, Integer> salaryByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getSalary,
                        Collectors.summingInt(Integer::intValue)
                )
        ));
```

Direct version:

```java
Map<String, Integer> salaryByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summingInt(Employee::getSalary)
        ));
```

---

# 10. `collectingAndThen()` Fundamentals ⭐⭐⭐⭐⭐

`collectingAndThen()` performs a downstream collection and then applies a finishing function.

Example:

```java
List<String> names = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.mapping(
                        Employee::getName,
                        Collectors.toList()
                ),
                List::copyOf
        ));
```

Mental model:

```text
Stream
  ↓
Collector
  ↓
Intermediate result
  ↓
Finisher function
  ↓
Final result
```

---

# 11. `collectingAndThen()` + `toSet()`

Create an unmodifiable Set:

```java
Set<String> names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.collectingAndThen(
                Collectors.toSet(),
                Collections::unmodifiableSet
        ));
```

---

# 12. `collectingAndThen()` + `maxBy()` 🔥🔥🔥

Find highest-paid employee and unwrap the Optional:

```java
Employee highestPaid = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                ),
                Optional::orElseThrow
        ));
```

This is a classic interview use case.

Without `collectingAndThen()`, `maxBy()` returns:

```java
Optional<Employee>
```

With `collectingAndThen()`, the finisher can convert it directly to:

```java
Employee
```

---

# 13. Highest-Paid Employee Per Department 🔥🔥🔥

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

Mental model:

```text
Employee
   ↓
groupBy(department)
   ↓
maxBy(salary)
   ↓
Optional<Employee>
   ↓
collectingAndThen
   ↓
Employee
```

---

# 14. `collectingAndThen()` + `toList()` → List Size

```java
Map<String, Integer> employeeCount = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.toList(),
                        List::size
                )
        ));
```

Result:

```text
IT       → 3
HR       → 2
Finance  → 1
```

`Collectors.counting()` is simpler when you only need the count, but this demonstrates the finisher concept.

---

# 15. `collectingAndThen()` + Sorting

Department → sorted employee names:

```java
Map<String, List<String>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.collectingAndThen(
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        ),
                        list -> list.stream()
                                .sorted()
                                .toList()
                )
        ));
```

---

# 16. `mapping()` vs `map()` ⭐⭐⭐⭐⭐

### `map()`

Transforms elements in the main stream:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .toList();
```

### `mapping()`

Transforms elements inside a downstream collector:

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

### Interview line

> `map()` is a Stream intermediate operation; `mapping()` is a Collector that adapts elements before passing them to another downstream collector.

---

# 17. `joining()` vs `toList()`

Need a List:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .toList();
```

Need one formatted String:

```java
String names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining(", "));
```

---

# 18. `collectingAndThen()` vs `map()`

`map()` transforms each stream element:

```java
stream.map(Employee::getName)
```

`collectingAndThen()` transforms the **completed collector result**:

```java
stream.collect(Collectors.collectingAndThen(
        Collectors.toList(),
        List::copyOf
));
```

Key distinction:

```text
map()                 → element-level transformation
collectingAndThen()   → result-level finishing transformation
```

---

# 19. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;

public class JoiningMappingCollectingAndThenDemo {

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

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000),
                new Employee(2, "Rahul", "IT", 120000),
                new Employee(3, "Amit", "IT", 180000),
                new Employee(4, "Priya", "HR", 130000),
                new Employee(5, "Ravi", "HR", 90000),
                new Employee(6, "Sneha", "Finance", 110000)
        );

        // 1. joining()
        String names = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.joining(", "));

        // 2. joining() with prefix and suffix
        String formattedNames = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.joining(", ", "[", "]"));

        // 3. Custom joining format
        String employeeSummary = employees.stream()
                .map(e -> e.getName() + "(" + e.getSalary() + ")")
                .collect(Collectors.joining(" | "));

        // 4. Department -> List of names
        Map<String, List<String>> namesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toList()
                        )
                ));

        // 5. Department -> Set of names
        Map<String, Set<String>> uniqueNamesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.toSet()
                        )
                ));

        // 6. Department -> joined names
        Map<String, String> joinedNamesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.mapping(
                                Employee::getName,
                                Collectors.joining(", ")
                        )
                ));

        // 7. Department -> average salary
        Map<String, Double> averageSalaryByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.averagingInt(Employee::getSalary)
                ));

        // 8. collectingAndThen -> immutable List
        List<String> immutableNames = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.collectingAndThen(
                        Collectors.toList(),
                        List::copyOf
                ));

        // 9. collectingAndThen -> unmodifiable Set
        Set<String> immutableDepartments = employees.stream()
                .map(Employee::getDepartment)
                .collect(Collectors.collectingAndThen(
                        Collectors.toSet(),
                        Collections::unmodifiableSet
                ));

        // 10. Highest-paid employee overall
        Employee highestPaid = employees.stream()
                .collect(Collectors.collectingAndThen(
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        ),
                        Optional::orElseThrow
                ));

        // 11. Highest-paid employee per department
        Map<String, Employee> highestPaidByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.collectingAndThen(
                                Collectors.maxBy(
                                        Comparator.comparingInt(Employee::getSalary)
                                ),
                                Optional::orElseThrow
                        )
                ));

        // 12. Department -> employee count using collectingAndThen
        Map<String, Integer> employeeCount = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.collectingAndThen(
                                Collectors.toList(),
                                List::size
                        )
                ));

        // 13. Department -> sorted employee names
        Map<String, List<String>> sortedNamesByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.collectingAndThen(
                                Collectors.mapping(
                                        Employee::getName,
                                        Collectors.toList()
                                ),
                                list -> list.stream().sorted().toList()
                        )
                ));

        System.out.println("Names: " + names);
        System.out.println("Formatted names: " + formattedNames);
        System.out.println("Employee summary: " + employeeSummary);
        System.out.println("Names by department: " + namesByDepartment);
        System.out.println("Unique names: " + uniqueNamesByDepartment);
        System.out.println("Joined names: " + joinedNamesByDepartment);
        System.out.println("Average salary: " + averageSalaryByDepartment);
        System.out.println("Immutable names: " + immutableNames);
        System.out.println("Immutable departments: " + immutableDepartments);
        System.out.println("Highest paid: " + highestPaid);
        System.out.println("Highest paid by department: " + highestPaidByDepartment);
        System.out.println("Employee count: " + employeeCount);
        System.out.println("Sorted names by department: " + sortedNamesByDepartment);
    }
}
```

---

# 20. Interview Scenarios 🔥🔥🔥

## Scenario 1 — Join all employee names

```java
String result = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.joining(", "));
```

---

## Scenario 2 — Department → Names

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

## Scenario 3 — Department → comma-separated names

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

## Scenario 4 — Highest salary overall without exposing Optional

```java
Employee result = employees.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                ),
                Optional::orElseThrow
        ));
```

---

## Scenario 5 — Highest salary per department ⭐⭐⭐⭐⭐

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

---

# 21. 25 Interview Questions 🎯

1. What is `Collectors.joining()`?
2. What are the three `joining()` overloads?
3. Can `joining()` work directly with arbitrary objects?
4. Why do we usually use `map()` before `joining()`?
5. What is `Collectors.mapping()`?
6. Difference between `map()` and `mapping()`?
7. Why is `mapping()` useful with `groupingBy()`?
8. Can `mapping()` be combined with `toSet()`?
9. Can `mapping()` be combined with `joining()`?
10. Can `mapping()` be combined with `counting()`?
11. What does `collectingAndThen()` do?
12. What is a downstream collector?
13. What is a finisher function?
14. How can `collectingAndThen()` unwrap an `Optional`?
15. How do you get the highest-paid employee per department?
16. How do you create an immutable result using `collectingAndThen()`?
17. Difference between `map()` and `collectingAndThen()`?
18. Difference between `joining()` and `toList()`?
19. How do you join values with prefix and suffix?
20. How do you group employees and join their names?
21. How do you group employees and create unique names using `mapping()`?
22. How do you sort a downstream collected List using `collectingAndThen()`?
23. Why does `maxBy()` return `Optional`?
24. When is `collectingAndThen()` unnecessary?
25. Explain `groupingBy + mapping + joining` in 2 minutes.

---

# 22. Coding Challenges 💻

### Challenge 1
Join all employee names using `", "`.

### Challenge 2
Join employee names with `[` and `]`.

### Challenge 3
Create department → comma-separated employee names.

### Challenge 4
Create department → unique employee names.

### Challenge 5
Create department → sorted employee names.

### Challenge 6
Create department → employee count using `collectingAndThen()`.

### Challenge 7
Find highest-paid employee using `maxBy()` + `collectingAndThen()`.

### Challenge 8 ⭐⭐⭐⭐⭐
Find highest-paid employee per department using:

```text
groupingBy
+ maxBy
+ collectingAndThen
```

### Challenge 9
Create an immutable List of employee names.

### Challenge 10
Create an unmodifiable Set of departments.

### Challenge 11
Create department → joined employee names ordered alphabetically.

### Challenge 12
Create department → employee names in uppercase joined by `" | "`.

### Challenge 13
Create department → unique names joined by `", "`.

### Challenge 14
Create department → sorted unique names.

### Challenge 15 — 5-Year Interview Level 🔥🔥🔥
Given customer transactions, group them by customer and produce:

```text
Customer → "TXN1:amount | TXN2:amount | TXN3:amount"
```

using `groupingBy()`, `mapping()` and `joining()`.

### Challenge 16
Create an immutable `Map<String, List<String>>` result after grouping.

### Challenge 17
Group employees by department and return only their names as an immutable List.

### Challenge 18
Find the maximum salary per department and return only the salary value.

### Challenge 19
Group products by category and join unique product names.

### Challenge 20 — Production Scenario 🔥🔥🔥
Build a response string for each department containing employee name, role and salary, formatted consistently with `mapping()` + `joining()`.

---

# 23. Common Mistakes ❌

### ❌ Mistake 1 — Confusing `map()` and `mapping()`

```text
map()       → Stream operation
mapping()   → Collector/downstream operation
```

### ❌ Mistake 2 — Forgetting `map()` before `joining()`

For objects:

```java
.map(Employee::getName)
.collect(Collectors.joining(", "))
```

### ❌ Mistake 3 — Thinking `collectingAndThen()` transforms every element

It transforms the **completed collector result**.

### ❌ Mistake 4 — Overusing `collectingAndThen()`

If a direct collector or stream operation already solves the problem clearly, prefer the simpler solution.

### ❌ Mistake 5 — Ignoring the Optional from `maxBy()` / `minBy()`

`maxBy()` returns `Optional<T>` because the stream may be empty.

### ❌ Mistake 6 — Using `mapping()` when normal `map()` is simpler

For a flat stream transformation, prefer `map()`.

---

# 24. Final Revision Sheet 🧠

```text
joining()
────────────────────
Combine CharSequence values into one String
```

```text
joining(", ")
────────────────────
Combine with delimiter
```

```text
joining(", ", "[", "]")
────────────────────
Delimiter + prefix + suffix
```

```text
mapping()
────────────────────
Transform elements before downstream collector
```

```text
groupingBy(key, mapping(value, downstream))
────────────────────
Group → transform → collect
```

```text
collectingAndThen(downstream, finisher)
────────────────────
Collect → post-process result
```

### Golden Rules

```text
Need one String?                       → joining()
Need transformation in a collector?   → mapping()
Need post-processing after collect?    → collectingAndThen()
Group + names?                          → groupingBy + mapping
Group + joined names?                  → groupingBy + mapping + joining
Max result without Optional?           → maxBy + collectingAndThen
```

---

# 25. 2-Minute Interview Script 🎤

> “`Collectors.joining()` combines CharacterSequence elements into a single String and supports delimiter, prefix and suffix. `Collectors.mapping()` is a downstream collector that transforms elements before passing them to another collector, so it is especially useful with `groupingBy()`. For example, I can group employees by department and then collect only their names using `groupingBy()` plus `mapping()`. I can replace `toList()` with `joining()` when I need a formatted String. `collectingAndThen()` performs a downstream collection first and then applies a finishing function to the collected result. A common example is `maxBy()`, which returns an Optional; `collectingAndThen()` can unwrap it with `Optional::orElseThrow`. So the key distinction is: `map()` transforms stream elements, `mapping()` adapts elements inside a collector, and `collectingAndThen()` transforms the final collector result.”

---

# 🧪 Complete Practice Code

[GitHub — 9.17 `joining()` / `mapping()` / `collectingAndThen()` Deep Dive Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/17-Collectors-joining-mapping-collectingAndThen-Deep-Dive)

---

## Navigation

[← 9.16 — `toMap()` / `toSet()` / `toList()` Deep Dive](../16-Collectors-toMap-toSet-toList-Deep-Dive/README.md)

**Current → 9.17 — `joining()` / `mapping()` / `collectingAndThen()` → ✅ Completed**

**Next → 9.18 — `Collectors.summarizingInt()` / `averagingInt()` / `summingInt()`**