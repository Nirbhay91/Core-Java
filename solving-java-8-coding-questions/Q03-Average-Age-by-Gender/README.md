# Q03 — Find Average Age of Male and Female Employees

## 🎯 Problem

Given a list of `Employee` objects, find the **average age separately for male and female employees**.

Example:

```text
Input:
Amit    → Male   → 30
Rahul   → Male   → 40
Priya   → Female → 25
Neha    → Female → 35
```

Expected output:

```text
Male   → 35.0
Female → 30.0
```

---

# 1. Requirement Breakdown

Requirement hai:

```text
Employee List
     ↓
Group by Gender
     ↓
Calculate Average Age
     ↓
Male Average + Female Average
```

Java 8 mein natural pattern:

```java
groupingBy() + averagingInt()
```

---

# 2. Java 8 Solution

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.averagingInt(Employee::getAge)
        ));
```

Example result:

```text
{
    Male=35.0,
    Female=30.0
}
```

---

# 3. Understand `groupingBy()` + `averagingInt()`

Ye Q1 ke pattern ka next level hai.

Q1 mein:

```java
groupingBy() + counting()
```

Use hua tha.

Q3 mein:

```java
groupingBy() + averagingInt()
```

Use hoga.

Memory:

```text
Category-wise count
→ groupingBy() + counting()

Category-wise average
→ groupingBy() + averagingInt()/averagingDouble()
```

---

# 4. Line-by-Line Explanation

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
Collectors.groupingBy(Employee::getGender, ...)
```

Employees ko gender ke according groups mein divide karta hai.

```text
Male
 ├── Amit 30
 └── Rahul 40

Female
 ├── Priya 25
 └── Neha 35
```

### Step 3 — `averagingInt()`

```java
Collectors.averagingInt(Employee::getAge)
```

Har gender group ke age values ka average calculate karta hai.

Male:

```text
(30 + 40) / 2 = 35.0
```

Female:

```text
(25 + 35) / 2 = 30.0
```

---

# 5. Complete Flow

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
      Ages: 30,40               Ages: 25,35
          ↓                         ↓
  averagingInt(age)        averagingInt(age)
          ↓                         ↓
         35.0                      30.0
          └────────────┬────────────┘
                       ↓
              Map<String, Double>
```

---

# 6. Dry Run

Input:

```text
Amit    → Male   → 30
Rahul   → Male   → 40
Priya   → Female → 25
Neha    → Female → 35
```

After grouping:

```text
Male   → [30, 40]
Female → [25, 35]
```

### Male

```text
sum = 30 + 40 = 70
count = 2
average = 70 / 2 = 35.0
```

### Female

```text
sum = 25 + 35 = 60
count = 2
average = 60 / 2 = 30.0
```

Final:

```java
{
    Male=35.0,
    Female=30.0
}
```

---

# 7. Why `averagingInt()`?

`Employee::getAge` returns an `int`.

Therefore:

```java
averagingInt(Employee::getAge)
```

is the natural collector.

It produces a `Double` average.

So result type is:

```java
Map<String, Double>
```

Not:

```java
Map<String, Integer>
```

because an average can contain a fractional value.

Example:

```text
30 + 31 + 32 = 93
93 / 3 = 31.0
```

Another example:

```text
30 + 31 = 61
61 / 2 = 30.5
```

---

# 8. `averagingInt()` vs `averagingLong()` vs `averagingDouble()`

### `averagingInt()`

Use when mapper returns `int` / `Integer`:

```java
averagingInt(Employee::getAge)
```

### `averagingLong()`

Use when mapper returns `long` / `Long`:

```java
averagingLong(Employee::getEmployeeId)
```

### `averagingDouble()`

Use when mapper returns `double` / `Double`:

```java
averagingDouble(Employee::getSalary)
```

Memory:

```text
int    → averagingInt()
long   → averagingLong()
double → averagingDouble()
```

---

# 9. Alternative — `filter()` + `average()`

Because the requirement specifically has only two genders, we can also do:

```java
OptionalDouble maleAverage = employeeList.stream()
        .filter(e -> "Male".equals(e.getGender()))
        .mapToInt(Employee::getAge)
        .average();

OptionalDouble femaleAverage = employeeList.stream()
        .filter(e -> "Female".equals(e.getGender()))
        .mapToInt(Employee::getAge)
        .average();
```

This works, but it performs separate stream traversals.

For a generic category-wise average:

```java
groupingBy() + averagingInt()
```

is cleaner.

---

# 10. Why `mapToInt()`?

This alternative:

```java
.mapToInt(Employee::getAge)
```

creates an `IntStream`.

Then:

```java
.average()
```

calculates the average.

Important difference:

```text
Stream<Employee>
      ↓
mapToInt()
      ↓
IntStream
      ↓
average()
      ↓
OptionalDouble
```

Whereas:

```java
groupingBy(
    Employee::getGender,
    averagingInt(Employee::getAge)
)
```

returns the grouped averages directly as:

```java
Map<String, Double>
```

---

# 11. Why Does `average()` Return `OptionalDouble`?

Consider:

```java
employeeList.stream()
        .filter(e -> "Male".equals(e.getGender()))
        .mapToInt(Employee::getAge)
        .average();
```

What if there are no male employees?

There is no value to average.

So Java returns:

```java
OptionalDouble
```

instead of simply returning `0`.

This avoids treating `0` as a real average.

Example:

```java
OptionalDouble average = ...;

if (average.isPresent()) {
    System.out.println(average.getAsDouble());
}
```

Or:

```java
average.ifPresent(System.out::println);
```

---

# 12. Important Difference — Empty Group

With the `groupingBy() + averagingInt()` solution, only groups that actually occur are present.

For example:

```text
Input:
Amit → Male → 30
Rahul → Male → 40
```

Result:

```text
{Male=35.0}
```

There may be no `Female` key at all.

If the business requirement says:

```text
Female → 0.0
```

you need to add that behavior explicitly after collecting.

---

# 13. Complete Interview-Ready Code

```java
public static Map<String, Double> averageAgeByGender(
        List<Employee> employeeList) {

    if (employeeList == null || employeeList.isEmpty()) {
        return Collections.emptyMap();
    }

    return employeeList.stream()
            .filter(Objects::nonNull)
            .filter(e -> e.getGender() != null)
            .collect(Collectors.groupingBy(
                    Employee::getGender,
                    Collectors.averagingInt(Employee::getAge)
            ));
}
```

This version explicitly ignores:

```text
null Employee
null Gender
```

If your domain guarantees those fields are non-null, the filters can be omitted.

---

# 14. What If Age Is `Integer` and Can Be Null?

Suppose:

```java
private Integer age;
```

Then:

```java
Employee::getAge
```

may return `null`.

You need to decide whether to:

```text
ignore employee
```

or:

```text
reject invalid data
```

or:

```text
provide a default
```

For example, ignoring missing ages:

```java
employeeList.stream()
        .filter(e -> e != null)
        .filter(e -> e.getGender() != null)
        .filter(e -> e.getAge() != null)
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.averagingInt(Employee::getAge)
        ));
```

---

# 15. Real-World Generalization

This pattern is extremely important for Java 8 interviews.

### Average salary by department

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

### Average age by gender

```java
groupingBy(gender, averagingInt(age))
```

### Average salary by gender

```java
groupingBy(gender, averagingDouble(salary))
```

The pattern remains:

```text
GROUP BY field
        ↓
AGGREGATE numeric field
```

---

# 16. Q1 → Q2 → Q3 Pattern

These first three questions are intentionally building one Java 8 pattern.

### Q1

```text
Gender-wise COUNT
↓
groupingBy() + counting()
```

### Q2

```text
Unique departments
↓
map() + distinct()
```

### Q3

```text
Gender-wise AVERAGE age
↓
groupingBy() + averagingInt()
```

This is how you should learn Streams—not by memorizing 50 separate programs.

---

# 17. Interview Question — Why Not `reduce()`?

You can calculate averages manually with `reduce()`, but it is more complicated because an average needs both:

```text
sum
count
```

A dedicated collector already knows this requirement:

```java
averagingInt()
```

So prefer the purpose-built collector.

---

# 18. Interview Question — `averagingInt()` vs `summarizingInt()`

### `averagingInt()`

Only average:

```java
Double
```

### `summarizingInt()`

Provides multiple statistics:

```text
count
sum
min
average
max
```

Example:

```java
Map<String, IntSummaryStatistics> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.summarizingInt(Employee::getAge)
        ));
```

Then:

```java
result.get("Male").getAverage();
```

Use `summarizingInt()` when you need multiple statistics from the same numeric field.

---

# 19. Interview Question — Can We Find Average and Count Together?

Yes. Use `summarizingInt()`:

```java
Map<String, IntSummaryStatistics> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.summarizingInt(Employee::getAge)
        ));
```

For male employees:

```java
IntSummaryStatistics stats = result.get("Male");

stats.getCount();
stats.getSum();
stats.getAverage();
stats.getMin();
stats.getMax();
```

This is a strong Java 8 follow-up.

---

# 20. Time Complexity

Let:

```text
N = number of employees
```

Each employee is processed once:

```text
Time = O(N)
```

If `K` is the number of gender groups:

```text
Space = O(K)
```

For two genders, the result map is effectively constant-sized, but the general complexity is better stated as `O(K)`.

---

# 21. Dry Run With Odd Average

Input:

```text
Male:
25
30
32
```

Calculation:

```text
25 + 30 + 32 = 87
87 / 3 = 29.0
```

Another example:

```text
25
30
```

```text
55 / 2 = 27.5
```

This is why the result must be capable of representing decimal values.

---

# 22. 🧠 Average Formula

genui{"learning_viz":{"type_id":"ARITHMETIC_MEAN","locale_override":"en-IN"}}

Conceptually:

```text
Average = Sum of ages / Number of employees
```

Java's `averagingInt()` handles this aggregation for us.

---

# 23. 2-Minute Interview Answer

> **"For average age by gender, I would use `Collectors.groupingBy()` with `Employee::getGender` as the classifier and `Collectors.averagingInt(Employee::getAge)` as the downstream collector. This first groups employees by gender and then calculates the average age within each group. The result is a `Map<String, Double>`. I use `averagingInt()` because age is an integer field, but the average itself can be fractional, so the result is Double. The time complexity is O(N), where N is the number of employees, and the result map requires O(K) space for K gender groups. If I also needed count, sum, min and max, I would use `summarizingInt()` instead."**

---

# 24. 30-Second Hinglish Answer

> **"Gender-wise average age nikalna hai, so main `groupingBy()` ke saath `averagingInt()` use karunga. `Employee::getGender` group key hoga aur `Employee::getAge` se average calculate hoga. Result `Map<String, Double>` milega, jaise `Male=35.0, Female=30.0`. `averagingInt()` use karne ka reason hai ki age int hai, but average decimal ho sakta hai. Time O(N) aur space O(K) hai. Agar count, sum, min, max bhi chahiye ho to `summarizingInt()` use kar sakte hain."**

---

# 25. 🧠 Memory Trick

```text
Gender-wise
     ↓
GROUP BY
     ↓
Age ka AVERAGE
     ↓
groupingBy()
     +
averagingInt()
```

### One-line rule

> **"Group-wise integer average → `groupingBy()` + `averagingInt()`"**

---

# 26. Common Interview Mistakes

### ❌ Mistake 1

Returning `Map<String, Integer>`.

Average can be decimal, so use:

```java
Map<String, Double>
```

### ❌ Mistake 2

Using `averagingDouble()` without explaining why.

For `int age`, `averagingInt()` is the natural choice.

### ❌ Mistake 3

Ignoring the empty-group case.

### ❌ Mistake 4

Confusing `average()` with `averagingInt()`.

```text
IntStream.average()       → OptionalDouble
Collectors.averagingInt() → Double
```

### ❌ Mistake 5

Using `reduce()` unnecessarily when a dedicated collector exists.

---

# 27. Follow-Up Questions

1. Why does `averagingInt()` return `Double`?
2. Difference between `averagingInt()` and `averagingDouble()`?
3. Why does `IntStream.average()` return `OptionalDouble`?
4. What happens if there are no female employees?
5. What is `summarizingInt()`?
6. How do you calculate average salary by department?
7. How do you get average and count together?
8. Can you solve it without Streams?
9. What is the time complexity?
10. What happens if age is null?
11. What is a downstream collector?
12. Why use `groupingBy()` instead of separate filters?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

Previous: [Q02 — Print Unique Departments](../Q02-Print-Unique-Departments/README.md)

## Status

✅ **Q03 Completed**

Next: **Q04 — Find the highest-paid employee**
