# Q07 — Find the Average Salary of Each Department

## 🎯 Problem

Given a list of `Employee` objects, find the **average salary of employees in each department**.

Example:

```text
Amit   → IT       → 60000
Rahul  → HR       → 50000
Priya  → IT       → 80000
Neha   → Finance  → 70000
Karan  → HR       → 70000
Rohit  → IT       → 100000
```

Expected output:

```text
IT       → 80000.0
HR       → 60000.0
Finance  → 70000.0
```

---

# 1. Core Approach

Requirement ko break karo:

```text
Employee List
      ↓
    stream()
      ↓
 groupingBy(department)
      ↓
 averagingDouble(salary)
      ↓
Map<Department, Average Salary>
```

Main Java 8 pattern:

```java
groupingBy() + averagingDouble()
```

---

# 2. Java 8 Solution

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

Example result:

```text
{
    IT=80000.0,
    HR=60000.0,
    Finance=70000.0
}
```

---

# 3. Line-by-Line Explanation

### Step 1 — `stream()`

```java
employeeList.stream()
```

Converts the employee list into a Stream.

```text
List<Employee>
      ↓
Stream<Employee>
```

### Step 2 — `groupingBy()`

```java
Collectors.groupingBy(Employee::getDepartment, ...)
```

Employees ko department ke according groups mein divide karta hai.

Conceptually:

```text
IT
 ├── Amit   → 60000
 ├── Priya  → 80000
 └── Rohit  → 100000

HR
 ├── Rahul  → 50000
 └── Karan  → 70000

Finance
 └── Neha   → 70000
```

### Step 3 — `averagingDouble()`

```java
Collectors.averagingDouble(Employee::getSalary)
```

Har department ke salary values ka average calculate karta hai.

```text
IT      → (60000 + 80000 + 100000) / 3 = 80000
HR      → (50000 + 70000) / 2 = 60000
Finance → 70000 / 1 = 70000
```

---

# 4. Complete Flow

```text
                 Employee List
                       ↓
                    stream()
                       ↓
             groupingBy(department)
                       ↓
          ┌────────────┼────────────┐
          ↓            ↓            ↓
          IT           HR         Finance
          ↓            ↓            ↓
   averagingDouble  averagingDouble  averagingDouble
          ↓            ↓            ↓
        80000        60000         70000
                       ↓
             Map<String, Double>
```

---

# 5. Dry Run

Input:

```text
Amit   → IT       → 60000
Rahul  → HR       → 50000
Priya  → IT       → 80000
Neha   → Finance  → 70000
Karan  → HR       → 70000
Rohit  → IT       → 100000
```

### IT

```text
60000 + 80000 + 100000 = 240000
240000 / 3 = 80000
```

### HR

```text
50000 + 70000 = 120000
120000 / 2 = 60000
```

### Finance

```text
70000 / 1 = 70000
```

Final:

```text
IT       → 80000.0
HR       → 60000.0
Finance  → 70000.0
```

---

# 6. Why `averagingDouble()`?

Because salary is represented as a numeric value and we need the **average** for each group.

Java provides specialized averaging collectors:

```java
averagingInt()
averagingLong()
averagingDouble()
```

For a salary represented as `double`, use:

```java
Collectors.averagingDouble(Employee::getSalary)
```

---

# 7. Why Is the Result `Map<String, Double>`?

The grouping key is:

```java
Employee::getDepartment
```

which is a `String`.

The downstream collector:

```java
averagingDouble(...)
```

returns a `Double`.

Therefore:

```java
Map<String, Double>
```

---

# 8. Q6 vs Q7

These two questions are extremely important because the grouping logic is the same but the downstream operation changes.

### Q6 — Count Employees

```java
groupingBy(
    Employee::getDepartment,
    Collectors.counting()
)
```

Pattern:

```text
GROUP BY + COUNT
```

### Q7 — Average Salary

```java
groupingBy(
    Employee::getDepartment,
    Collectors.averagingDouble(Employee::getSalary)
)
```

Pattern:

```text
GROUP BY + AVG
```

### Memory Trick

```text
COUNT → counting()
AVG   → averagingDouble()
```

---

# 9. SQL Connection

SQL version:

```sql
SELECT department, AVG(salary)
FROM employee
GROUP BY department;
```

Java 8 equivalent:

```java
employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

Think:

```text
SQL GROUP BY
      ↓
Java groupingBy()

SQL AVG()
      ↓
Java averagingDouble()
```

---

# 10. `averagingInt()` vs `averagingLong()` vs `averagingDouble()`

### Integer field

```java
Collectors.averagingInt(Employee::getAge)
```

### Long field

```java
Collectors.averagingLong(Employee::getExperienceInMonths)
```

### Double field

```java
Collectors.averagingDouble(Employee::getSalary)
```

Important:

> Choose the collector based on the numeric type of the value being averaged.

---

# 11. If Salary Is an `int`

Suppose:

```java
private int salary;
```

Then:

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

Notice that the average is still a `Double`.

For example:

```text
50000 + 70000 = 120000
120000 / 2 = 60000.0
```

---

# 12. Why Not `mapToDouble().average()` Directly?

You could first group employees and then calculate averages:

```java
Map<String, List<Employee>> grouped = employeeList.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));

Map<String, Double> result = grouped.entrySet().stream()
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> e.getValue().stream()
                        .mapToDouble(Employee::getSalary)
                        .average()
                        .orElse(0.0)
        ));
```

This works, but it is more verbose and creates intermediate employee lists.

Preferred Java 8 solution:

```java
groupingBy(
    Employee::getDepartment,
    averagingDouble(Employee::getSalary)
)
```

---

# 13. What Happens with an Empty Department Group?

Normally `groupingBy()` only creates groups for employees that actually exist, so an empty department does not appear in the result.

If a department must always appear even when there are no employees, that becomes a separate business requirement and needs explicit handling.

---

# 14. Null Employee Handling

If the list may contain null employees:

```java
Map<String, Double> result = employeeList.stream()
        .filter(Objects::nonNull)
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

This prevents:

```text
NullPointerException
```

---

# 15. Null Department Handling

If department can be null, decide the business rule.

For example, normalize null to `UNKNOWN`:

```java
Map<String, Double> result = employeeList.stream()
        .filter(Objects::nonNull)
        .collect(Collectors.groupingBy(
                e -> e.getDepartment() == null
                        ? "UNKNOWN"
                        : e.getDepartment(),
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

Now:

```text
null department → UNKNOWN
```

If null departments should be ignored instead:

```java
.filter(e -> e.getDepartment() != null)
```

---

# 16. Null Salary Handling

If salary is a primitive:

```java
private double salary;
```

it cannot be null.

If salary is a wrapper:

```java
private Double salary;
```

then define the business rule for null salaries before calculating the average.

For example, ignoring null salaries requires a custom downstream approach rather than blindly calling `averagingDouble(Employee::getSalary)`.

A common interview answer is:

> **"I would clarify whether a null salary should be ignored, treated as zero, or considered invalid, because each produces a different business result."**

---

# 17. Department with Highest Average Salary

Natural follow-up:

> Which department has the highest average salary?

First calculate averages:

```java
Map<String, Double> avgSalaryByDepartment = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

Then:

```java
Optional<Map.Entry<String, Double>> result = avgSalaryByDepartment.entrySet()
        .stream()
        .max(Map.Entry.comparingByValue());
```

This combines:

```text
groupingBy
   ↓
averagingDouble
   ↓
max
```

---

# 18. Sort Departments by Average Salary

If interviewer asks:

> Sort departments by average salary descending.

```java
List<Map.Entry<String, Double>> result = avgSalaryByDepartment.entrySet()
        .stream()
        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
        .collect(Collectors.toList());
```

Example:

```text
IT       → 100000.0
Finance  → 85000.0
HR       → 65000.0
```

Sorting adds:

```text
O(K log K)
```

where K is the number of departments.

---

# 19. Department-Wise Minimum Salary

Use `summarizingDouble()` if you need multiple statistics, or `minBy()` if you specifically need the employee with minimum salary.

For the minimum salary value:

```java
Map<String, Optional<Double>> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.mapping(
                        Employee::getSalary,
                        Collectors.minBy(Double::compare)
                )
        ));
```

But if the requirement is only the average, keep the solution simple with `averagingDouble()`.

---

# 20. `summarizingDouble()` — Advanced Follow-Up

If interviewer asks for **count, sum, min, max and average salary together**, `summarizingDouble()` is useful:

```java
Map<String, DoubleSummaryStatistics> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.summarizingDouble(Employee::getSalary)
        ));
```

Then for a department:

```java
DoubleSummaryStatistics stats = result.get("IT");

stats.getCount();
stats.getSum();
stats.getMin();
stats.getMax();
stats.getAverage();
```

Memory:

```text
Need only average
→ averagingDouble()

Need multiple statistics
→ summarizingDouble()
```

---

# 21. `averagingDouble()` vs `summarizingDouble()`

### Only average

```java
averagingDouble(Employee::getSalary)
```

Result:

```text
Double
```

### Complete statistics

```java
summarizingDouble(Employee::getSalary)
```

Result:

```text
DoubleSummaryStatistics
```

which provides:

```text
count
sum
min
max
average
```

---

# 22. Department + Average Salary + Employee Count

A useful advanced interview question is:

> For each department, give both employee count and average salary.

You can use `collectingAndThen()` with a custom result, or create a small DTO/record for the response.

For interview discussion, the key idea is:

```text
groupingBy(department)
        ↓
multiple aggregations
        ↓
count + average
```

For maintainable production code, a dedicated DTO is usually clearer than returning nested generic maps.

---

# 23. Why Not Calculate Average Manually?

Manual approach:

```java
Map<String, Double> sum = new HashMap<>();
Map<String, Integer> count = new HashMap<>();

for (Employee employee : employeeList) {
    String department = employee.getDepartment();
    sum.merge(department, employee.getSalary(), Double::sum);
    count.merge(department, 1, Integer::sum);
}
```

Then calculate:

```text
average = sum / count
```

It works, but Java 8's downstream collector directly expresses the requirement:

```java
averagingDouble(Employee::getSalary)
```

For interview readability, prefer the direct collector when appropriate.

---

# 24. Floating-Point / Money Caveat

For interview coding, `double` is often used because the question usually expects a numeric average.

For production financial calculations, be careful with binary floating-point representation.

If exact monetary arithmetic is required, consider:

```java
BigDecimal
```

and define the required rounding mode/scale explicitly.

Do not blindly say:

> "double is always correct for money."

It is not.

---

# 25. Time Complexity

Let:

```text
N = number of employees
K = number of departments
```

Every employee is processed once for grouping and aggregation.

Expected time:

```text
O(N)
```

Result map contains one value per department:

```text
Space = O(K)
```

If you additionally sort the department averages:

```text
O(K log K)
```

is added.

---

# 26. Q6 → Q7 Pattern

This is one of the most important Java 8 interview patterns.

### Q6

```text
Department-wise employee count

GROUP BY + COUNT
        ↓
groupingBy() + counting()
```

### Q7

```text
Department-wise average salary

GROUP BY + AVG
        ↓
groupingBy() + averagingDouble()
```

### General Pattern

```text
GROUP BY + aggregation
          ↓
    groupingBy()
          +
 downstream collector
```

Examples:

```text
COUNT → counting()
AVG   → averagingDouble()
SUM   → summingDouble()
MIN   → minBy()
MAX   → maxBy()
STATS → summarizingDouble()
```

---

# 27. Common Interview Variations

### Variation 1

> Average salary by department.

```java
groupingBy + averagingDouble
```

### Variation 2

> Department with highest average salary.

```text
groupingBy + averagingDouble + max
```

### Variation 3

> Sort departments by average salary.

```text
groupingBy + averagingDouble + sorted
```

### Variation 4

> Department-wise total salary.

```java
Collectors.summingDouble(Employee::getSalary)
```

### Variation 5

> Department-wise salary statistics.

```java
Collectors.summarizingDouble(Employee::getSalary)
```

---

# 28. Interview-Ready Code

```java
public static Map<String, Double> averageSalaryByDepartment(
        List<Employee> employeeList) {

    if (employeeList == null || employeeList.isEmpty()) {
        return Collections.emptyMap();
    }

    return employeeList.stream()
            .filter(Objects::nonNull)
            .filter(e -> e.getDepartment() != null)
            .collect(Collectors.groupingBy(
                    Employee::getDepartment,
                    Collectors.averagingDouble(Employee::getSalary)
            ));
}
```

This version assumes null departments should be ignored. Adjust that rule if the business requirement says null should be represented as `UNKNOWN`.

---

# 29. 2-Minute Interview Answer

> **"For department-wise average salary, I would use `Collectors.groupingBy()` with `Employee::getDepartment` as the classifier and `Collectors.averagingDouble(Employee::getSalary)` as the downstream collector. `groupingBy()` creates one group per department, and `averagingDouble()` calculates the average salary for each group. The result is a `Map<String, Double>`. The expected time complexity is O(N), where N is the number of employees, and the result map uses O(K) additional space for K departments. If I also need count, sum, min and max along with average, I would consider `summarizingDouble()` instead."**

---

# 30. 30-Second Hinglish Answer

> **"Department-wise average salary chahiye, so main `groupingBy()` ke saath `averagingDouble()` use karunga. `Employee::getDepartment` group key hoga aur `Employee::getSalary` ka average har group ke liye calculate hoga. Result `Map<String, Double>` milega. Time complexity O(N) aur result map ke liye O(K) space hai. Simple rule: `GROUP BY + AVG` means `groupingBy() + averagingDouble()`."**

---

# 31. 🧠 Memory Trick

```text
Department-wise
       ↓
     GROUP BY
       ↓
      AVG
       ↓
groupingBy()
      +
averagingDouble()
```

### One-line rule

> **"Group-wise average → `groupingBy()` + `averagingDouble()`"**

---

# 32. Common Interview Mistakes

### ❌ Mistake 1

Using `counting()` when the requirement asks for average.

### ❌ Mistake 2

Forgetting that `averagingDouble()` returns `Double`.

### ❌ Mistake 3

Grouping into `List<Employee>` and manually calculating averages when a downstream collector is sufficient.

### ❌ Mistake 4

Ignoring null employees or null departments when the domain permits them.

### ❌ Mistake 5

Using `double` blindly for production-grade financial calculations.

### ❌ Mistake 6

Sorting all employees when only department averages are required.

---

# 33. Follow-Up Questions

1. What is `averagingDouble()`?
2. Difference between `averagingInt()`, `averagingLong()` and `averagingDouble()`?
3. Why is the result `Map<String, Double>`?
4. Difference between Q6 and Q7?
5. How do you find the department with the highest average salary?
6. How do you sort departments by average salary?
7. How do you calculate total salary by department?
8. How do you get count, sum, min, max and average together?
9. What is `summarizingDouble()`?
10. How do you handle null salary?
11. How do you handle null department?
12. What is the time and space complexity?
13. How does this map to SQL `GROUP BY + AVG()`?
14. Why might `BigDecimal` be preferable for monetary calculations?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

Previous: [Q06 — Count Employees by Department](../Q06-Count-Employees-by-Department/README.md)

## Status

✅ **Q07 Completed**

Next: **Q08 — Find the youngest male employee**
