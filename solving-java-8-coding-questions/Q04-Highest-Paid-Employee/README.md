# Q04 — Find the Highest-Paid Employee

## 🎯 Problem

Given a list of `Employee` objects, find the employee who has the **highest salary**.

Example:

```text
Amit   → 50000
Rahul  → 75000
Priya  → 65000
Neha   → 90000
```

Expected output:

```text
Neha → 90000
```

---

# 1. Approach

Requirement ko simple steps mein break karo:

```text
Employee List
     ↓
   stream()
     ↓
Compare salaries
     ↓
   max()
     ↓
Highest-paid Employee
```

Java 8 mein direct approach:

```java
stream() + max() + Comparator.comparingDouble()
```

---

# 2. Java 8 Solution

If salary is `double`:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .max(Comparator.comparingDouble(Employee::getSalary));
```

Because `max()` may have no result when the list is empty, it returns:

```java
Optional<Employee>
```

---

# 3. Complete Example

```java
Optional<Employee> highestPaid = employeeList.stream()
        .max(Comparator.comparingDouble(Employee::getSalary));

highestPaid.ifPresent(employee ->
        System.out.println(employee.getName() + " → " + employee.getSalary())
);
```

Output:

```text
Neha → 90000.0
```

---

# 4. Line-by-Line Explanation

### Step 1 — `stream()`

```java
employeeList.stream()
```

Converts:

```text
List<Employee>
```

to:

```text
Stream<Employee>
```

---

### Step 2 — `max()`

```java
.max(...)
```

Stream ke elements mein se maximum element find karta hai according to the supplied `Comparator`.

---

### Step 3 — `Comparator.comparingDouble()`

```java
Comparator.comparingDouble(Employee::getSalary)
```

Comparator ko salary ke basis par define karta hai.

Conceptually:

```text
Employee A → salary 50000
Employee B → salary 75000
Employee C → salary 90000
                     ↑
                   MAX
```

---

# 5. Complete Flow

```text
                 List<Employee>
                       ↓
                    stream()
                       ↓
             comparingDouble(salary)
                       ↓
                      max()
                       ↓
              Optional<Employee>
                       ↓
                Highest-paid
```

---

# 6. Dry Run

Input:

```text
Amit   → 50000
Rahul  → 75000
Priya  → 65000
Neha   → 90000
```

Comparison:

```text
50000 vs 75000 → 75000
75000 vs 65000 → 75000
75000 vs 90000 → 90000
```

Final:

```text
Neha → 90000
```

---

# 7. Why `max()` Instead of `sorted()`?

A common interview question.

You could write:

```java
employeeList.stream()
        .sorted(Comparator.comparingDouble(Employee::getSalary).reversed())
        .findFirst();
```

But this sorts the entire stream just to find one employee.

Better:

```java
.max(Comparator.comparingDouble(Employee::getSalary))
```

Because we only need the maximum value, not the complete sorted list.

### Complexity

`max()`:

```text
O(N)
```

Sorting:

```text
O(N log N)
```

So `max()` is the correct choice for this requirement.

---

# 8. Why Does `max()` Return `Optional<Employee>`?

Suppose:

```java
List<Employee> employeeList = Collections.emptyList();
```

There is no highest-paid employee.

Java therefore returns:

```java
Optional.empty()
```

instead of returning `null`.

Example:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .max(Comparator.comparingDouble(Employee::getSalary));

highestPaid.ifPresent(System.out::println);
```

---

# 9. Avoid `get()` Without Checking

This is risky:

```java
highestPaid.get();
```

If the Optional is empty, it throws:

```text
NoSuchElementException
```

Prefer:

```java
highestPaid.ifPresent(...);
```

or:

```java
Employee employee = highestPaid.orElse(null);
```

depending on the application contract.

---

# 10. If Salary Is `int`

If:

```java
int getSalary()
```

you can use:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .max(Comparator.comparingInt(Employee::getSalary));
```

If salary is `long`:

```java
Comparator.comparingLong(Employee::getSalary)
```

If salary is `double`:

```java
Comparator.comparingDouble(Employee::getSalary)
```

Memory trick:

```text
int    → comparingInt()
long   → comparingLong()
double → comparingDouble()
```

---

# 11. Why `comparingDouble()`?

Suppose salary is:

```java
private double salary;
```

Then:

```java
Comparator.comparingDouble(Employee::getSalary)
```

is preferable because it creates a comparator based on the primitive `double` value.

It clearly communicates that salary is a numeric field.

---

# 12. Alternative — `reduce()`

You can also solve it using `reduce()`:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .reduce((e1, e2) ->
                e1.getSalary() > e2.getSalary() ? e1 : e2
        );
```

This works, but `max()` is more expressive.

Why?

Because the requirement literally says:

```text
Find maximum
```

So:

```java
max(comparator)
```

communicates the intent directly.

---

# 13. Alternative — `sorted()` + `findFirst()`

```java
Optional<Employee> highestPaid = employeeList.stream()
        .sorted(Comparator.comparingDouble(Employee::getSalary).reversed())
        .findFirst();
```

Correct, but unnecessarily expensive for a single maximum.

Use it only when you actually need the sorted stream/result.

---

# 14. Highest Salary Only

If interviewer asks:

> Find only the highest salary, not the employee.

You can use:

```java
OptionalDouble highestSalary = employeeList.stream()
        .mapToDouble(Employee::getSalary)
        .max();
```

Notice the difference:

```text
Need Employee object
→ Stream<Employee>.max(comparator)

Need salary value only
→ mapToDouble().max()
```

---

# 15. `OptionalDouble` vs `Optional<Employee>`

### Highest Employee

```java
Optional<Employee>
```

### Highest Salary

```java
OptionalDouble
```

Because the first operation returns an object, while the second operates on a primitive numeric stream.

---

# 16. What If Multiple Employees Have the Same Highest Salary?

Example:

```text
Amit  → 90000
Neha  → 90000
```

`max()` returns one employee according to the reduction/encounter behavior; it does not return all ties.

If interviewer asks:

> Return all employees with the highest salary.

Use two steps.

### Step 1 — Find maximum salary

```java
OptionalDouble maxSalary = employeeList.stream()
        .mapToDouble(Employee::getSalary)
        .max();
```

### Step 2 — Filter employees with that salary

```java
List<Employee> result = employeeList.stream()
        .filter(e -> e.getSalary() == maxSalary.orElse(Double.NaN))
        .collect(Collectors.toList());
```

For production code, consider the numeric type and equality semantics carefully; for money, `BigDecimal` is usually preferable to `double`.

---

# 17. Money — Important Production Point

For interview coding, salary may be represented as `double`.

But in real financial/payment systems, avoid using `double` for exact monetary calculations.

Prefer:

```java
BigDecimal salary;
```

Then:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .max(Comparator.comparing(Employee::getSalary));
```

This is an important senior-level follow-up.

---

# 18. If Salary Is `BigDecimal`

Example:

```java
class Employee {
    private BigDecimal salary;

    public BigDecimal getSalary() {
        return salary;
    }
}
```

Then:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .max(Comparator.comparing(Employee::getSalary));
```

No `comparingDouble()` is needed because `BigDecimal` already implements `Comparable`.

---

# 19. Null Salary Handling

If salary can be null, decide the business rule.

For example, treat null salary as lower than every real salary:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .filter(Objects::nonNull)
        .max(Comparator.comparing(
                Employee::getSalary,
                Comparator.nullsFirst(Comparator.naturalOrder())
        ));
```

If null salary should make the record invalid, filter or validate it before the calculation.

Do not silently invent business semantics in production code.

---

# 20. Null Employee Handling

If the list itself may contain null elements:

```java
Optional<Employee> highestPaid = employeeList.stream()
        .filter(Objects::nonNull)
        .max(Comparator.comparingDouble(Employee::getSalary));
```

Again, ideally the domain model should define whether null employees are allowed.

---

# 21. Generic Pattern — Find Maximum by Any Field

This is not only for salary.

### Highest age

```java
employeeList.stream()
        .max(Comparator.comparingInt(Employee::getAge));
```

### Highest bonus

```java
employeeList.stream()
        .max(Comparator.comparingDouble(Employee::getBonus));
```

### Latest joining date

```java
employeeList.stream()
        .max(Comparator.comparing(Employee::getJoiningDate));
```

Pattern:

```text
stream()
  ↓
max()
  ↓
Comparator.comparing...(field)
```

---

# 22. Q3 → Q4 Java 8 Pattern

Q3:

```text
Group by gender
      ↓
Average age
      ↓
groupingBy() + averagingInt()
```

Q4:

```text
Compare salary
      ↓
Find maximum employee
      ↓
max() + Comparator
```

This is an important shift:

```text
Aggregation → Collector
Selection   → Terminal max()
```

---

# 23. Time Complexity

Let:

```text
N = number of employees
```

`max()` compares each employee against the current maximum.

Therefore:

```text
Time = O(N)
```

Additional space for the sequential stream calculation is:

```text
O(1)
```

apart from the returned `Optional` and stream implementation overhead.

This is better than sorting:

```text
Sorting → O(N log N)
max()   → O(N)
```

---

# 24. Dry Run

Suppose:

```text
Amit   → 50000
Rahul  → 75000
Priya  → 65000
Neha   → 90000
```

Current maximum:

```text
Amit → 50000
```

Compare Rahul:

```text
75000 > 50000
→ Rahul becomes max
```

Compare Priya:

```text
65000 < 75000
→ Rahul remains max
```

Compare Neha:

```text
90000 > 75000
→ Neha becomes max
```

Final:

```text
Neha → 90000
```

---

# 25. Interview Question — Why Is `max()` O(N)?

Because finding a maximum requires checking every candidate at least once in the general case.

Conceptually:

```text
N employees
     ↓
N - 1 comparisons
     ↓
Maximum
```

So:

```text
O(N)
```

---

# 26. Interview Question — Why Not `collect(toList())` First?

Unnecessary:

```java
List<Employee> employees = employeeList.stream()
        .collect(Collectors.toList());
```

Then finding max adds another step.

The original list is already available.

Directly use:

```java
employeeList.stream()
        .max(...);
```

Keep the pipeline focused on the actual requirement.

---

# 27. Interview Question — What Is `Comparator` Doing?

`Comparator` tells `max()`:

> **"Employee ko kis property ke basis par compare karna hai?"**

Here:

```java
Comparator.comparingDouble(Employee::getSalary)
```

means:

```text
Compare Employee objects
        ↓
using salary
```

Without a comparator, Java would need a natural ordering for `Employee`.

---

# 28. `Comparable` vs `Comparator` Follow-Up

### Comparable

Object ke natural ordering ko define karta hai:

```java
class Employee implements Comparable<Employee> {
    @Override
    public int compareTo(Employee other) {
        return Double.compare(this.salary, other.salary);
    }
}
```

Then:

```java
employeeList.stream().max(Comparator.naturalOrder());
```

### Comparator

External/custom ordering define karta hai:

```java
Comparator.comparingDouble(Employee::getSalary)
```

Interview memory:

> **Comparable = class ke andar natural ordering.**
> **Comparator = bahar se comparison strategy.**

---

# 29. Interview-Ready Code

```java
public static Optional<Employee> findHighestPaid(
        List<Employee> employeeList) {

    if (employeeList == null || employeeList.isEmpty()) {
        return Optional.empty();
    }

    return employeeList.stream()
            .filter(Objects::nonNull)
            .max(Comparator.comparingDouble(Employee::getSalary));
}
```

Usage:

```java
findHighestPaid(employeeList)
        .ifPresent(e ->
                System.out.println(e.getName() + " → " + e.getSalary())
        );
```

---

# 30. 2-Minute Interview Answer

> **"To find the highest-paid employee, I would use the Stream `max()` terminal operation with a comparator based on salary. For example, `employeeList.stream().max(Comparator.comparingDouble(Employee::getSalary))`. `max()` returns `Optional<Employee>` because the list can be empty. The time complexity is O(N), because every employee needs to be considered, and the additional space is O(1) for the sequential calculation. I would not sort the complete list because sorting would take O(N log N) while I only need the maximum. If salary is monetary data in a real system, I would prefer `BigDecimal` rather than `double`."**

---

# 31. 30-Second Hinglish Answer

> **"Highest-paid employee nikalne ke liye main Stream ka `max()` use karunga aur salary ke basis par `Comparator.comparingDouble(Employee::getSalary)` dunga. Result `Optional<Employee>` hoga because list empty bhi ho sakti hai. Iska time O(N) hai aur sequential calculation ke liye extra space O(1) hai. `sorted()` use nahi karunga because complete sorting O(N log N) hogi, jabki hume sirf maximum chahiye. Real project mein salary money hai to `double` ki jagah `BigDecimal` prefer karunga."**

---

# 32. 🧠 Memory Trick

```text
Highest / Maximum object
        ↓
      max()
        ↓
   Comparator
        ↓
     field
```

### One-line rule

> **"Kisi object ka highest value wala record chahiye → `max()` + `Comparator`."**

---

# 33. Common Interview Mistakes

### ❌ Mistake 1

Sorting the entire list when only maximum is required.

### ❌ Mistake 2

Using:

```java
employeeList.stream().max(...).get();
```

without handling an empty list.

### ❌ Mistake 3

Confusing:

```text
highest employee
```

with:

```text
highest salary
```

### ❌ Mistake 4

Using `==` with floating-point monetary values for equality.

### ❌ Mistake 5

Using `double` for money in a production financial domain without discussing precision.

### ❌ Mistake 6

Not knowing why `Comparator` is required.

---

# 34. Follow-Up Questions

1. Why does `max()` return `Optional`?
2. Why not use `sorted()`?
3. What is the time complexity of `max()`?
4. How would you find the highest salary only?
5. How would you return all employees having the highest salary?
6. Difference between `Comparable` and `Comparator`?
7. Difference between `comparingInt`, `comparingLong`, and `comparingDouble`?
8. How would you handle null salary?
9. Why should money usually use `BigDecimal`?
10. How would you find the second-highest-paid employee?
11. How would you find the top 3 highest-paid employees?
12. Can this be solved using `reduce()`?
13. Why is `max()` O(N) but sorting O(N log N)?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

Previous: [Q03 — Average Age by Gender](../Q03-Average-Age-by-Gender/README.md)

## Status

✅ **Q04 Completed**

Next: **Q05 — Find employees who joined after 2015**
