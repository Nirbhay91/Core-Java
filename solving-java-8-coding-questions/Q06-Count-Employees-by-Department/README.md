# Q06 — Count Employees in Each Department

## 🎯 Problem

Given a list of `Employee` objects, find the **number of employees in each department**.

Example:

```text
Amit   → IT
Rahul  → HR
Priya  → IT
Neha   → Finance
Karan  → HR
Rohit  → IT
```

Expected output:

```text
IT       → 3
HR       → 2
Finance  → 1
```

---

# 1. Approach

Requirement ko break karo:

```text
Employee List
      ↓
    stream()
      ↓
 groupingBy(department)
      ↓
    counting()
      ↓
Map<Department, Count>
```

Java 8 ka direct pattern:

```java
groupingBy() + counting()
```

---

# 2. Java 8 Solution

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Example result:

```text
{
    IT=3,
    HR=2,
    Finance=1
}
```

---

# 3. Line-by-Line Explanation

### Step 1 — `stream()`

```java
employeeList.stream()
```

Employee list ko Stream mein convert karta hai.

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
 ├── Amit
 ├── Priya
 └── Rohit

HR
 ├── Rahul
 └── Karan

Finance
 └── Neha
```

### Step 3 — `counting()`

```java
Collectors.counting()
```

Har department group ke employees count karta hai.

```text
IT      → 3
HR      → 2
Finance → 1
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
      counting()   counting()   counting()
          ↓            ↓            ↓
          3            2            1
                       ↓
             Map<String, Long>
```

---

# 5. Dry Run

Input:

```text
Amit   → IT
Rahul  → HR
Priya  → IT
Neha   → Finance
Karan  → HR
Rohit  → IT
```

Process one by one:

```text
Amit   → IT      → IT = 1
Rahul  → HR      → HR = 1
Priya  → IT      → IT = 2
Neha   → Finance → Finance = 1
Karan  → HR      → HR = 2
Rohit  → IT      → IT = 3
```

Final:

```text
IT       → 3
HR       → 2
Finance  → 1
```

---

# 6. Why `groupingBy()`?

Requirement hai:

> **Department-wise count**

Iska SQL-style thinking:

```sql
SELECT department, COUNT(*)
FROM employee
GROUP BY department;
```

Java Stream equivalent:

```java
groupingBy(Employee::getDepartment, counting())
```

Memory trick:

> **GROUP BY field + COUNT → `groupingBy()` + `counting()`**

---

# 7. Why `counting()`?

`groupingBy()` sirf groups banata hai.

Agar hum sirf likhen:

```java
Map<String, List<Employee>> result = employeeList.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

to result hoga:

```text
IT       → [Amit, Priya, Rohit]
HR       → [Rahul, Karan]
Finance  → [Neha]
```

But hume employees ki lists nahi chahiye.

Hume sirf count chahiye.

Therefore:

```java
Collectors.counting()
```

use karte hain.

---

# 8. Why Does the Result Use `Long`?

Because:

```java
Collectors.counting()
```

returns:

```java
Long
```

Therefore:

```java
Map<String, Long>
```

not:

```java
Map<String, Integer>
```

This is a common Java 8 interview follow-up.

---

# 9. Q2 vs Q6

These questions look similar but have different requirements.

### Q2 — Unique Departments

Only department names chahiye:

```java
employeeList.stream()
        .map(Employee::getDepartment)
        .distinct()
        .forEach(System.out::println);
```

Pattern:

```text
map + distinct
```

### Q6 — Count Employees by Department

Department + employee count chahiye:

```java
employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Pattern:

```text
groupingBy + counting
```

### Important

> **Unique values → `distinct()`**
>
> **Group-wise count → `groupingBy()` + `counting()`**

---

# 10. Alternative — `groupingBy()` Without `counting()`

You can first create lists:

```java
Map<String, List<Employee>> grouped = employeeList.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Then:

```java
Map<String, Integer> result = grouped.entrySet().stream()
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> e.getValue().size()
        ));
```

But this creates employee lists unnecessarily.

Better:

```java
groupingBy(Employee::getDepartment, counting())
```

because the collector directly performs the aggregation.

---

# 11. Alternative — Manual `HashMap`

Without Streams:

```java
Map<String, Long> result = new HashMap<>();

for (Employee employee : employeeList) {
    result.merge(employee.getDepartment(), 1L, Long::sum);
}
```

This is also O(N) expected time.

But for a Java 8 Stream coding interview, `groupingBy()` + `counting()` demonstrates the intended collector pattern more clearly.

---

# 12. `merge()` Explained

In the manual solution:

```java
result.merge(employee.getDepartment(), 1L, Long::sum);
```

Meaning:

```text
If department doesn't exist:
    add department → 1

If department exists:
    oldCount + 1
```

Example:

```text
IT absent → IT = 1
IT exists → IT = 2
IT exists → IT = 3
```

This is a useful Java Collections follow-up.

---

# 13. Null Department Handling

If department is allowed to be null, define the business rule.

A clean approach is to normalize null to `UNKNOWN`:

```java
Map<String, Long> result = employeeList.stream()
        .filter(Objects::nonNull)
        .collect(Collectors.groupingBy(
                e -> e.getDepartment() == null
                        ? "UNKNOWN"
                        : e.getDepartment(),
                Collectors.counting()
        ));
```

Result can contain:

```text
UNKNOWN → 2
```

If null departments should be rejected or ignored, implement that explicitly instead.

---

# 14. Empty List

For:

```java
Collections.emptyList()
```

result will be:

```java
{}
```

The collector naturally returns an empty map.

---

# 15. Sorting Department Counts

Suppose interviewer asks:

> Sort departments by employee count descending.

First calculate counts:

```java
Map<String, Long> countByDepartment = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Then sort entries:

```java
List<Map.Entry<String, Long>> sorted = countByDepartment.entrySet()
        .stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
        .collect(Collectors.toList());
```

Example:

```text
IT       → 10
Sales    → 7
HR       → 4
Finance  → 2
```

Important:

> The original `groupingBy()` result is a map; sorting the entries is a separate requirement.

---

# 16. Top Department by Employee Count

If interviewer asks:

> Which department has the highest number of employees?

Use:

```java
Optional<Map.Entry<String, Long>> result = countByDepartment.entrySet()
        .stream()
        .max(Map.Entry.comparingByValue());
```

Or as a complete pipeline:

```java
Optional<Map.Entry<String, Long>> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ))
        .entrySet()
        .stream()
        .max(Map.Entry.comparingByValue());
```

This combines:

```text
groupingBy + counting + max
```

---

# 17. Department-Wise Average Salary

This is a natural follow-up:

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

Pattern:

```text
Department-wise count
→ groupingBy + counting

Department-wise average
→ groupingBy + averagingDouble
```

---

# 18. Department-Wise Highest Salary

Using a downstream `maxBy()` collector:

```java
Map<String, Optional<Employee>> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingDouble(Employee::getSalary)
                )
        ));
```

This is an important advanced Java 8 follow-up because it combines:

```text
groupingBy
   +
maxBy
   +
Comparator
```

---

# 19. `groupingBy()` Has a Downstream Collector

This syntax:

```java
groupingBy(classifier, downstreamCollector)
```

has two important parts.

### Classifier

```java
Employee::getDepartment
```

Answers:

> **"Employee ko kis key/group mein daalna hai?"**

### Downstream collector

```java
Collectors.counting()
```

Answers:

> **"Har group ke andar kya calculation karni hai?"**

So:

```java
groupingBy(
    Employee::getDepartment,
    counting()
)
```

means:

> **Department ke according group karo, phir har group ko count karo.**

---

# 20. SQL vs Java Stream Thinking

A very useful interview connection:

### SQL

```sql
SELECT department, COUNT(*)
FROM employee
GROUP BY department;
```

### Java 8

```java
employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Think of:

```text
SQL GROUP BY
       ↓
Java groupingBy()

SQL COUNT
       ↓
Java counting()
```

---

# 21. `groupingBy()` vs `partitioningBy()`

### `groupingBy()`

Multiple arbitrary categories:

```text
IT
HR
Finance
Sales
```

### `partitioningBy()`

Boolean split:

```text
true
false
```

Example:

```java
employeeList.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getSalary() > 100000
        ));
```

For department counts:

```java
groupingBy()
```

is the natural choice.

---

# 22. Case Sensitivity

These values are different Strings:

```text
IT
it
It
```

So `groupingBy(Employee::getDepartment)` creates separate keys.

If department names should be case-insensitive, normalize first:

```java
Map<String, Long> result = employeeList.stream()
        .filter(e -> e.getDepartment() != null)
        .collect(Collectors.groupingBy(
                e -> e.getDepartment().toUpperCase(Locale.ROOT),
                Collectors.counting()
        ));
```

Now:

```text
IT
it
It
```

can all map to:

```text
IT
```

---

# 23. Preserve Department Ordering

If you need departments in encounter order:

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                LinkedHashMap::new,
                Collectors.counting()
        ));
```

This uses:

```java
LinkedHashMap
```

as the map supplier.

If sorted keys are required, use an appropriate sorted map such as `TreeMap`.

---

# 24. Time Complexity

Let:

```text
N = number of employees
K = number of departments
```

Each employee is processed once.

Expected time with hash-based grouping:

```text
O(N)
```

Result map stores one entry per department:

```text
Space = O(K)
```

If we additionally sort the departments by count:

```text
O(K log K)
```

is added for sorting the `K` department entries.

---

# 25. Why Is It Not O(N × K)?

Because we don't compare every employee with every department.

The hash-based map gives expected constant-time key lookup/update.

Conceptually:

```text
Employee 1 → hash department → update count
Employee 2 → hash department → update count
Employee 3 → hash department → update count
...
```

Therefore expected:

```text
O(N)
```

---

# 26. Q2 vs Q6 vs Q7 Pattern

These questions form a useful progression.

### Q2

```text
Unique departments
↓
map + distinct
```

### Q6

```text
Department-wise count
↓
groupingBy + counting
```

### Next-level follow-up

```text
Department-wise average salary
↓
groupingBy + averagingDouble
```

The main skill is recognizing the requirement and choosing the correct downstream collector.

---

# 27. Common Interview Variations

### Variation 1

> Print department-wise count.

```java
groupingBy + counting
```

### Variation 2

> Find department with maximum employees.

```text
groupingBy + counting + max
```

### Variation 3

> Sort departments by employee count.

```text
groupingBy + counting + sorted
```

### Variation 4

> Find average salary by department.

```text
groupingBy + averagingDouble
```

### Variation 5

> Find highest-paid employee in each department.

```text
groupingBy + maxBy
```

---

# 28. Interview-Ready Code

```java
public static Map<String, Long> countEmployeesByDepartment(
        List<Employee> employeeList) {

    if (employeeList == null || employeeList.isEmpty()) {
        return Collections.emptyMap();
    }

    return employeeList.stream()
            .filter(Objects::nonNull)
            .filter(e -> e.getDepartment() != null)
            .collect(Collectors.groupingBy(
                    Employee::getDepartment,
                    Collectors.counting()
            ));
}
```

This version assumes null departments should be ignored. If the business rule says otherwise, normalize them to `UNKNOWN` instead.

---

# 29. 2-Minute Interview Answer

> **"For department-wise employee count, I would use `Collectors.groupingBy()` with `Employee::getDepartment` as the classifier and `Collectors.counting()` as the downstream collector. `groupingBy()` creates one group for each department and `counting()` calculates the number of employees in each group. The result is a `Map<String, Long>`. The expected time complexity is O(N), where N is the number of employees, and additional space is O(K), where K is the number of distinct departments. I prefer this over grouping into `List<Employee>` and then calling `size()`, because that would retain employee lists unnecessarily when I only need counts."**

---

# 30. 30-Second Hinglish Answer

> **"Department-wise employee count chahiye, so main `groupingBy()` ke saath `counting()` use karunga. `Employee::getDepartment` group key hoga aur `counting()` har department ke employees count karega. Result `Map<String, Long>` milega, jaise `IT=3, HR=2`. Time complexity expected O(N) aur extra map space O(K) hai. Simple rule: SQL mein `GROUP BY department + COUNT(*)`, Java Stream mein `groupingBy() + counting()`."**

---

# 31. 🧠 Memory Trick

```text
Department-wise
       ↓
     GROUP BY
       ↓
     COUNT
       ↓
groupingBy()
      +
counting()
```

### One-line rule

> **"Group-wise count → `groupingBy()` + `counting()`"**

---

# 32. Common Interview Mistakes

### ❌ Mistake 1

Using `Map<String, Integer>` with `Collectors.counting()`.

Correct:

```java
Map<String, Long>
```

### ❌ Mistake 2

Using `map() + distinct()` when the requirement asks for counts.

### ❌ Mistake 3

Grouping into `List<Employee>` and then calling `size()` when only counts are required.

### ❌ Mistake 4

Forgetting null department handling when the domain allows null.

### ❌ Mistake 5

Assuming map iteration order without choosing an appropriate map implementation.

### ❌ Mistake 6

Not knowing what the downstream collector means.

---

# 33. Follow-Up Questions

1. What does `groupingBy()` do?
2. What is a downstream collector?
3. Why is the result `Map<String, Long>`?
4. Why not `Map<String, Integer>`?
5. Difference between Q2 unique departments and Q6 department-wise count?
6. How do you find the department with the maximum employees?
7. How do you sort departments by employee count?
8. How do you find average salary by department?
9. How do you find the highest-paid employee in each department?
10. Difference between `groupingBy()` and `partitioningBy()`?
11. How do you preserve department insertion order?
12. What is the time and space complexity?
13. How would you handle null department?
14. How does this map to SQL `GROUP BY`?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

Previous: [Q05 — Employees Joined After 2015](../Q05-Employees-Joined-After-2015/README.md)

## Status

✅ **Q06 Completed**

Next: **Q07 — Find the average salary of each department**
