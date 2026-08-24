# Q01 — Count Male and Female Employees

## 🎯 Problem

Given a list of `Employee` objects, find how many **male** and **female** employees are present in the organization.

Example:

```text
Input:
Employee(..., "Male", ...)
Employee(..., "Female", ...)
Employee(..., "Male", ...)

Output:
Male   → 2
Female → 1
```

---

# 1. Approach

Hume employee list ko gender ke basis par group karna hai aur har group ka count nikalna hai.

Think like this:

```text
Employee List
     ↓
   stream()
     ↓
 groupingBy(gender)
     ↓
  counting()
     ↓
Map<String, Long>
```

Java 8 mein is problem ke liye best direct combination hai:

```java
groupingBy() + counting()
```

---

# 2. Java 8 Solution

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.counting()
        ));
```

Example result:

```text
{
    Male=5,
    Female=3
}
```

---

# 3. Code Ko Line-by-Line Samjho

### Step 1 — `employeeList.stream()`

```java
employeeList.stream()
```

Employee list ko Stream mein convert karta hai.

```text
List<Employee>
      ↓
Stream<Employee>
```

---

### Step 2 — `collect()`

```java
.collect(...)
```

Stream ke processed elements ko ek final result mein collect karta hai.

Yahan result ek `Map` hoga.

---

### Step 3 — `groupingBy()`

```java
Collectors.groupingBy(Employee::getGender, ...)
```

Ye employee ko uske gender ke according groups mein divide karega.

Conceptually:

```text
Male
 ├── Employee 1
 ├── Employee 3
 └── Employee 5

Female
 ├── Employee 2
 └── Employee 4
```

---

### Step 4 — `counting()`

```java
Collectors.counting()
```

Har group ke employees ko count karega.

Final:

```text
Male   → 3
Female → 2
```

---

# 4. Complete Flow

```text
                 Employee List
                       ↓
                    stream()
                       ↓
              groupingBy(gender)
                       ↓
          ┌────────────┴────────────┐
          ↓                         ↓
        Male                      Female
          ↓                         ↓
      counting()                counting()
          ↓                         ↓
          5                         3
          └────────────┬────────────┘
                       ↓
              Map<String, Long>
```

---

# 5. Dry Run

Suppose employees hain:

```text
1. Amit    → Male
2. Priya   → Female
3. Rahul   → Male
4. Neha    → Female
5. Karan   → Male
```

### Employee 1

```text
Male → 1
```

### Employee 2

```text
Female → 1
```

### Employee 3

```text
Male → 2
```

### Employee 4

```text
Female → 2
```

### Employee 5

```text
Male → 3
```

Final result:

```java
{
    Male=3,
    Female=2
}
```

---

# 6. Why `Long` and Not `Integer`?

`Collectors.counting()` returns:

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

This is a common interview follow-up.

---

# 7. Alternative — `toMap()` / Manual Counting

Without `groupingBy()` we can manually maintain a map:

```java
Map<String, Long> result = new HashMap<>();

for (Employee employee : employeeList) {
    result.merge(employee.getGender(), 1L, Long::sum);
}
```

This works, but for a Java 8 Stream interview question:

```java
groupingBy() + counting()
```

is more expressive and directly communicates the requirement.

---

# 8. Alternative — `groupingBy()` Without `counting()`

You could first create groups:

```java
Map<String, List<Employee>> grouped = employeeList.stream()
        .collect(Collectors.groupingBy(Employee::getGender));
```

Then count:

```java
Map<String, Integer> result = grouped.entrySet().stream()
        .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> e.getValue().size()
        ));
```

But this unnecessarily creates lists when we only need counts.

Better:

```java
groupingBy(Employee::getGender, counting())
```

---

# 9. Why Not `filter()` Twice?

Another possible solution:

```java
long maleCount = employeeList.stream()
        .filter(e -> "Male".equals(e.getGender()))
        .count();

long femaleCount = employeeList.stream()
        .filter(e -> "Female".equals(e.getGender()))
        .count();
```

This is valid, but it scans the list separately for each gender.

For a general grouping requirement, prefer:

```java
groupingBy() + counting()
```

because the intent is clearer and one stream traversal can build all groups.

---

# 10. Why Use `"Male".equals(...)`?

Prefer:

```java
"Male".equals(employee.getGender())
```

over:

```java
employee.getGender().equals("Male")
```

because if `getGender()` returns `null`, the first version is safe while the second can throw `NullPointerException`.

Also never use:

```java
employee.getGender() == "Male"
```

for content comparison.

`==` compares object references for `String`, while `.equals()` compares content.

---

# 11. What If Gender Is Null?

With:

```java
groupingBy(Employee::getGender, Collectors.counting())
```

you should consider the null-handling behavior of the collector/JDK implementation and your domain contract. In production code, it is usually better to define whether gender is mandatory or normalize unknown values before grouping.

For example:

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                e -> e.getGender() == null ? "UNKNOWN" : e.getGender(),
                Collectors.counting()
        ));
```

Then:

```text
UNKNOWN → 2
```

---

# 12. Time Complexity

Let:

```text
N = number of employees
```

Each employee is processed once for grouping/counting.

Therefore expected time:

```text
O(N)
```

The map stores one entry per distinct gender/category.

If `K` is the number of distinct gender values:

```text
Space = O(K)
```

For the typical two-category case:

```text
K = 2
```

so auxiliary space is effectively:

```text
O(1)
```

---

# 13. Interview Question — Why `groupingBy()`?

### Answer

> **"Because the requirement is to create groups based on gender and then calculate an aggregate count for each group. `groupingBy()` handles the grouping and `counting()` performs the aggregation."**

---

# 14. Interview Question — What Does `groupingBy()` Return?

In this case:

```java
Map<String, Long>
```

Because:

```text
Key   → gender
Value → number of employees
```

Conceptually:

```text
Map<Gender, Count>
```

---

# 15. Interview Question — `groupingBy()` vs `partitioningBy()`

### `groupingBy()`

Use when there can be multiple arbitrary groups:

```text
HR
IT
Sales
Finance
```

### `partitioningBy()`

Use when the result is a boolean split:

```text
true
false
```

Example:

```java
partitioningBy(e -> e.getAge() > 25)
```

For gender counting, `groupingBy()` is the natural choice.

---

# 16. Interview Question — Can We Use `counting()` Without `groupingBy()`?

Yes, but then it counts the entire stream:

```java
long total = employeeList.stream()
        .collect(Collectors.counting());
```

That gives total employees, not gender-wise counts.

So:

```text
counting()
   ↓
Total count

groupingBy() + counting()
   ↓
Category-wise count
```

---

# 17. Real-World Interpretation

This pattern is not limited to gender.

The same approach works for:

```text
Employees by department
Orders by status
Users by country
Tickets by priority
Products by category
Payments by status
```

Generic pattern:

```java
stream.collect(
    Collectors.groupingBy(
        classifier,
        Collectors.counting()
    )
);
```

---

# 18. Generalized Example — Employees by Department

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Result:

```text
IT       → 10
HR       → 4
Finance  → 6
Sales    → 8
```

Same pattern, different classifier.

---

# 19. Interview-Ready Code

```java
public static Map<String, Long> countByGender(List<Employee> employeeList) {
    if (employeeList == null || employeeList.isEmpty()) {
        return Collections.emptyMap();
    }

    return employeeList.stream()
            .collect(Collectors.groupingBy(
                    Employee::getGender,
                    Collectors.counting()
            ));
}
```

This version also handles a null/empty input contract explicitly.

---

# 20. 🏆 COMPLETE INTERVIEW PRACTICE CODE — COPY & WRITE YOURSELF

> **This is the one place to practice the complete answer from scratch.**
>
> First try writing it without looking at the solution. Then compare line-by-line.

## Step 1 — Employee Model

```java
public class Employee {

    private int id;
    private String name;
    private String gender;
    private double salary;

    public Employee(int id, String name, String gender, double salary) {
        this.id = id;
        this.name = name;
        this.gender = gender;
        this.salary = salary;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getGender() {
        return gender;
    }

    public double getSalary() {
        return salary;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", gender='" + gender + '\'' +
                ", salary=" + salary +
                '}';
    }
}
```

## Step 2 — Interview Solution Class

```java
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class CountMaleFemaleEmployees {

    public static Map<String, Long> countByGender(
            List<Employee> employeeList) {

        if (employeeList == null || employeeList.isEmpty()) {
            return Collections.emptyMap();
        }

        return employeeList.stream()
                .collect(Collectors.groupingBy(
                        Employee::getGender,
                        Collectors.counting()
                ));
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Amit", "Male", 70000),
                new Employee(2, "Priya", "Female", 80000),
                new Employee(3, "Rahul", "Male", 90000),
                new Employee(4, "Neha", "Female", 75000),
                new Employee(5, "Karan", "Male", 85000)
        );

        Map<String, Long> result = countByGender(employees);

        System.out.println("Gender-wise Employee Count:");
        result.forEach((gender, count) ->
                System.out.println(gender + " -> " + count));
    }
}
```

Expected output conceptually:

```text
Gender-wise Employee Count:
Male -> 3
Female -> 2
```

> `HashMap`/`groupingBy()` does not guarantee a specific iteration order, so do not make your interview answer depend on whether `Male` or `Female` prints first.

---

## Step 3 — Null-Safe Interview Variant

When the interviewer asks:

> **"What if gender is null?"**

Use an explicit normalization strategy:

```java
public static Map<String, Long> countByGender(
        List<Employee> employeeList) {

    if (employeeList == null || employeeList.isEmpty()) {
        return Collections.emptyMap();
    }

    return employeeList.stream()
            .collect(Collectors.groupingBy(
                    employee -> employee.getGender() == null
                            ? "UNKNOWN"
                            : employee.getGender(),
                    Collectors.counting()
            ));
}
```

This makes the business decision explicit instead of leaving null handling implicit.

---

## Step 4 — Manual Java 8 Alternative

If interviewer says:

> **"Don't use `groupingBy()`."**

```java
public static Map<String, Long> countByGender(
        List<Employee> employeeList) {

    Map<String, Long> result = new java.util.HashMap<>();

    if (employeeList == null) {
        return result;
    }

    for (Employee employee : employeeList) {
        result.merge(
                employee.getGender(),
                1L,
                Long::sum
        );
    }

    return result;
}
```

Interview point:

```text
groupingBy + counting
→ declarative Stream solution

merge
→ imperative/manual solution
```

---

## Step 5 — Traditional Loop Alternative

If interviewer says:

> **"Don't use Stream API at all."**

```java
public static Map<String, Long> countByGender(
        List<Employee> employeeList) {

    Map<String, Long> result = new java.util.HashMap<>();

    if (employeeList == null) {
        return result;
    }

    for (Employee employee : employeeList) {
        String gender = employee.getGender();

        if (gender == null) {
            gender = "UNKNOWN";
        }

        Long currentCount = result.get(gender);

        if (currentCount == null) {
            result.put(gender, 1L);
        } else {
            result.put(gender, currentCount + 1);
        }
    }

    return result;
}
```

---

## Step 6 — Interview Variations to Practice

### Variation A — Count only Male

```java
long maleCount = employeeList.stream()
        .filter(employee ->
                "Male".equals(employee.getGender()))
        .count();
```

### Variation B — Count only Female

```java
long femaleCount = employeeList.stream()
        .filter(employee ->
                "Female".equals(employee.getGender()))
        .count();
```

### Variation C — Group by Department

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

### Variation D — Gender + Average Salary

```java
Map<String, Double> averageSalaryByGender =
        employeeList.stream()
                .collect(Collectors.groupingBy(
                        Employee::getGender,
                        Collectors.averagingDouble(
                                Employee::getSalary)
                ));
```

### Variation E — Gender + Maximum Salary

```java
Map<String, Optional<Employee>> highestPaidByGender =
        employeeList.stream()
                .collect(Collectors.groupingBy(
                        Employee::getGender,
                        Collectors.maxBy(
                                Comparator.comparingDouble(
                                        Employee::getSalary)
                        )
                ));
```

### Variation F — Count by Gender and Department

```java
Map<String, Map<String, Long>> result =
        employeeList.stream()
                .collect(Collectors.groupingBy(
                        Employee::getGender,
                        Collectors.groupingBy(
                                Employee::getDepartment,
                                Collectors.counting()
                        )
                ));
```

---

# 21. Full Interview Script 🎤

### Interviewer:
> Count male and female employees using Java 8.

### You:

> **"Sure. Since the requirement is category-wise counting, I'll use a Stream with `Collectors.groupingBy()` and `Collectors.counting()`. The key will be `Employee::getGender`, and the downstream collector will count the employees in each group."**

Then write:

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.counting()
        ));
```

Then explain:

> **"`stream()` converts the list to a stream. `groupingBy()` creates groups by gender. `counting()` counts each group. Because `counting()` returns `Long`, the result type is `Map<String, Long>`. The expected time complexity is O(N)."**

Then proactively mention:

> **"If null or unknown gender is possible, I would normalize it to `UNKNOWN` based on the domain requirement."**

---

# 22. Rapid-Fire Follow-Up Practice

Try answering these without looking above:

```text
1. Why groupingBy()?
2. Why counting()?
3. Why Long?
4. Time complexity?
5. Space complexity?
6. What if gender is null?
7. What if Stream API is not allowed?
8. What if groupingBy() is not allowed?
9. What if I need average salary per gender?
10. What if I need highest-paid employee per gender?
11. groupingBy() vs partitioningBy()?
12. How does merge() work?
13. Why not filter twice?
14. Why not groupingBy() alone?
15. What is a downstream collector?
```

---

# 23. Final Interview Cheat Sheet 🧠

```text
Category-wise count
        ↓
groupingBy()
        ↓
counting()
        ↓
Map<String, Long>
```

```java
employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.counting()
        ));
```

### One-line memory formula

> **GROUP → `groupingBy()` | COUNT → `counting()` | RESULT → `Map<K, Long>`**

---

# 24. 2-Minute Interview Answer

> **"For this problem, I need category-wise counting. So I would stream the employee list and use `Collectors.groupingBy()` with `Employee::getGender` as the classifier and `Collectors.counting()` as the downstream collector. The result is a `Map<String, Long>` where the key is the gender and the value is the number of employees in that group. This processes the employees in one stream pipeline and has O(N) expected time complexity, where N is the number of employees. The extra space is O(K), where K is the number of distinct gender values. I prefer this over running separate filters for male and female because the requirement is naturally a grouping-and-aggregation problem. If the domain allows null gender values, I would explicitly normalize them to `UNKNOWN` or handle them according to the business contract."**

---

# 25. 30-Second Hinglish Answer

> **"Is problem mein mujhe gender-wise employee count chahiye, so main `groupingBy()` ke saath `counting()` use karunga. `Employee::getGender` group key hoga aur `counting()` har group ke employees count karega. Result `Map<String, Long>` milega, jaise `Male=5, Female=3`. Time complexity O(N) hai aur extra space O(K), jahan K distinct genders hain. Simple memory trick: `groupingBy = group banana`, `counting = count karna`."**

---

# 26. 🧠 Memory Trick

```text
Requirement:
"Gender-wise count"

          ↓

GROUP BY gender
          ↓

COUNT employees
          ↓

groupingBy() + counting()
```

### One-line rule

> **"Category-wise count chahiye → `groupingBy()` + `counting()`"**

---

# 27. Common Interview Mistakes

### ❌ Mistake 1

Using `==` for String comparison.

### ❌ Mistake 2

Returning `Map<String, Integer>` while using `Collectors.counting()`.

### ❌ Mistake 3

Using `groupingBy()` alone and creating `List<Employee>` when only counts are needed.

### ❌ Mistake 4

Running separate streams for every gender without explaining the trade-off.

### ❌ Mistake 5

Not explaining what `groupingBy()` and `counting()` individually do.

---

# 28. Follow-Up Questions

1. What does `Collectors.groupingBy()` return?
2. Why does `counting()` return `Long`?
3. Difference between `groupingBy()` and `partitioningBy()`?
4. Can you solve it without streams?
5. Can you count employees department-wise using the same pattern?
6. What is the time complexity?
7. What happens if the gender value is null?
8. How would you return the most common gender?
9. How would you group by gender and then calculate average salary?
10. What is a downstream collector?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

## Status

✅ **Q01 Completed**

Next: **Q02 — Print all unique departments**
