# Solving Java 8 Coding Questions — Employee Management System

> **Source / Practice Reference:** [JavaConceptOfTheDay — Solving Real Time Queries Using Java 8 Features](https://javaconceptoftheday.com/solving-real-time-queries-using-java-8-features-employee-management-system/)

This folder is for **Java 8 interview coding practice** using `Stream`, `Collectors`, `Comparator`, `Optional`, `groupingBy`, `partitioningBy`, `summarizingDouble`, `filter`, `map`, `sorted`, `min` and `max`.

The original article presents an Employee Management System and 15 real-time queries. citeturn1search0

---

## 📚 Problem Set

1. Count male and female employees
2. Print all unique departments
3. Find average age of male and female employees
4. Find the highest-paid employee
5. Find employees who joined after 2015
6. Count employees in each department
7. Find average salary of each department
8. Find youngest male employee in Product Development
9. Find the most experienced employee
10. Count male/female employees in Sales and Marketing
11. Find average salary of male and female employees
12. List employees in each department
13. Find average and total salary of the organization
14. Partition employees by age (>25 vs <=25)
15. Find the oldest employee, age and department

The 15 questions correspond to the problem set listed in the reference article. citeturn1search0turn1search1

---

## 🎯 Java 8 Concepts Covered

| Concept | Questions |
|---|---|
| `Stream.filter()` | 5, 8, 10 |
| `map()` | 2, 5 |
| `distinct()` | 2 |
| `groupingBy()` | 1, 6, 7, 10, 11, 12 |
| `counting()` | 1, 6, 10 |
| `averagingInt()` | 3 |
| `averagingDouble()` | 7, 11 |
| `maxBy()` | 4 |
| `min()` | 8 |
| `max()` | 15 |
| `sorted()` | 9 |
| `findFirst()` | 9 |
| `partitioningBy()` | 14 |
| `summarizingDouble()` | 13 |
| `Comparator.comparingInt()` | 8, 9, 15 |
| `Comparator.comparingDouble()` | 4 |
| `Optional` | 4, 8, 9, 15 |

---

# 👨‍💻 Employee Model

```java
public class Employee {

    private int id;
    private String name;
    private int age;
    private String gender;
    private String department;
    private int yearOfJoining;
    private double salary;

    public Employee(int id, String name, int age, String gender,
                    String department, int yearOfJoining, double salary) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.gender = gender;
        this.department = department;
        this.yearOfJoining = yearOfJoining;
        this.salary = salary;
    }

    public int getId() { return id; }
    public String getName() { return name; }
    public int getAge() { return age; }
    public String getGender() { return gender; }
    public String getDepartment() { return department; }
    public int getYearOfJoining() { return yearOfJoining; }
    public double getSalary() { return salary; }
}
```

---

# 1️⃣ Count Male and Female Employees

### Question

How many male and female employees are there in the organization?

### Approach

```text
employeeList
     ↓
stream()
     ↓
groupingBy(gender)
     ↓
counting()
```

### Java 8

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.counting()
        ));
```

### Key Concept

`groupingBy()` creates groups and `counting()` counts elements inside each group.

---

# 2️⃣ Print All Unique Departments

```java
employeeList.stream()
        .map(Employee::getDepartment)
        .distinct()
        .forEach(System.out::println);
```

### Flow

```text
Employee
 ↓
Department
 ↓
distinct()
 ↓
Print
```

---

# 3️⃣ Average Age of Male and Female Employees

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.averagingInt(Employee::getAge)
        ));
```

### Important

`averagingInt()` returns `Double` even though age is an integer.

---

# 4️⃣ Highest-Paid Employee

```java
Optional<Employee> result = employeeList.stream()
        .max(Comparator.comparingDouble(Employee::getSalary));
```

Or using collector:

```java
Optional<Employee> result = employeeList.stream()
        .collect(Collectors.maxBy(
                Comparator.comparingDouble(Employee::getSalary)
        ));
```

### Interview Point

Why `Optional`?

Because the list could be empty and there may be no maximum element.

---

# 5️⃣ Employees Who Joined After 2015

```java
employeeList.stream()
        .filter(e -> e.getYearOfJoining() > 2015)
        .map(Employee::getName)
        .forEach(System.out::println);
```

### Concepts

- `filter()` → selects employees
- `map()` → converts Employee to name
- `forEach()` → consumes the result

---

# 6️⃣ Employee Count in Each Department

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

### Memory Trick

> **Group by department + count employees.**

---

# 7️⃣ Average Salary of Each Department

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

---

# 8️⃣ Youngest Male Employee in Product Development

```java
Optional<Employee> result = employeeList.stream()
        .filter(e -> "Male".equals(e.getGender()))
        .filter(e -> "Product Development".equals(e.getDepartment()))
        .min(Comparator.comparingInt(Employee::getAge));
```

### Important Improvement

Prefer:

```java
"Male".equals(e.getGender())
```

over:

```java
e.getGender() == "Male"
```

because `==` compares references for objects, while `.equals()` compares string content.

---

# 9️⃣ Most Experienced Employee

The employee who joined earliest has the most experience.

```java
Optional<Employee> result = employeeList.stream()
        .min(Comparator.comparingInt(Employee::getYearOfJoining));
```

This is simpler than sorting the complete list and calling `findFirst()`.

### Why?

```text
min() → directly finds minimum
sorted() → sorts all elements first
```

So `min()` is preferable when only the minimum is required.

---

# 🔟 Male and Female Employees in Sales and Marketing

```java
Map<String, Long> result = employeeList.stream()
        .filter(e -> "Sales And Marketing".equals(e.getDepartment()))
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.counting()
        ));
```

### Pattern

```text
filter department
      ↓
group by gender
      ↓
count
```

---

# 1️⃣1️⃣ Average Salary of Male and Female Employees

```java
Map<String, Double> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getGender,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

---

# 1️⃣2️⃣ List Employees in Each Department

```java
Map<String, List<Employee>> result = employeeList.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Then:

```java
result.forEach((department, employees) -> {
    System.out.println("Department: " + department);

    employees.forEach(employee ->
            System.out.println(employee.getName())
    );
});
```

### Key Difference

```text
counting()       → Map<String, Long>
groupingBy only  → Map<String, List<Employee>>
```

---

# 1️⃣3️⃣ Average and Total Salary of Organization

```java
DoubleSummaryStatistics statistics = employeeList.stream()
        .collect(Collectors.summarizingDouble(Employee::getSalary));

System.out.println("Average = " + statistics.getAverage());
System.out.println("Total = " + statistics.getSum());
```

`DoubleSummaryStatistics` also provides:

```java
getCount()
getSum()
getMin()
getAverage()
getMax()
```

This is useful when multiple salary statistics are needed in one pass.

---

# 1️⃣4️⃣ Partition Employees by Age

Requirement:

```text
Age <= 25
vs
Age > 25
```

```java
Map<Boolean, List<Employee>> result = employeeList.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getAge() > 25
        ));
```

### Result Meaning

```text
true  → age > 25
false → age <= 25
```

### `groupingBy()` vs `partitioningBy()`

```text
Known two-way boolean split → partitioningBy()
Multiple arbitrary groups   → groupingBy()
```

---

# 1️⃣5️⃣ Oldest Employee

```java
Optional<Employee> result = employeeList.stream()
        .max(Comparator.comparingInt(Employee::getAge));
```

Then:

```java
result.ifPresent(employee -> {
    System.out.println(employee.getName());
    System.out.println(employee.getAge());
    System.out.println(employee.getDepartment());
});
```

---

# 🧠 Master Pattern Sheet

## Filtering

```java
stream.filter(condition)
```

Use when:

> **Mujhe sirf matching records chahiye.**

---

## Mapping

```java
stream.map(Employee::getName)
```

Use when:

> **Object ko kisi aur form mein convert karna hai.**

---

## Grouping

```java
Collectors.groupingBy(...)
```

Use when:

> **Category-wise data chahiye.**

---

## Counting

```java
Collectors.counting()
```

Use when:

> **Har group mein kitne records hain?**

---

## Average

```java
Collectors.averagingInt(...)
Collectors.averagingDouble(...)
```

---

## Maximum / Minimum

```java
stream.max(comparator)
stream.min(comparator)
```

---

## Partitioning

```java
Collectors.partitioningBy(predicate)
```

Use for:

```text
true / false
```

---

## Statistics

```java
Collectors.summarizingDouble(...)
```

Use when multiple statistics are required together.

---

# 🎯 Interview Preparation Strategy

Don't memorize 15 solutions separately.

Learn the patterns:

```text
Filter
 ↓
Group
 ↓
Count
 ↓
Average
 ↓
Min / Max
 ↓
Map
 ↓
Partition
 ↓
Statistics
```

Then derive the solution from the requirement.

### Example

Interviewer:

> Find average salary department-wise.

Think:

```text
Department-wise
      ↓
groupingBy(department)
      ↓
averagingDouble(salary)
```

Code:

```java
Collectors.groupingBy(
    Employee::getDepartment,
    Collectors.averagingDouble(Employee::getSalary)
)
```

---

# ⚠️ Important Corrections for Interview Code

The source article demonstrates some comparisons using `==` for `String` values. In Java interview/production code, use `.equals()` or the null-safe constant-first form instead:

```java
"Male".equals(e.getGender())
```

and:

```java
"Product Development".equals(e.getDepartment())
```

This repository intentionally uses the safer form.

---

# 🔗 Reference

Original problem set and Employee Management System example:

[JavaConceptOfTheDay — Solving Real Time Queries Using Java 8 Features](https://javaconceptoftheday.com/solving-real-time-queries-using-java-8-features-employee-management-system/)

The reference article contains the Employee model, sample employee list and 15 Java 8 queries. citeturn1search0

---

## Status

✅ **Java 8 Employee Management System — Added**

### Next recommended cycle

Solve these questions **one by one**, with:

```text
Question
→ Approach
→ Stream pipeline
→ Code
→ Dry Run
→ Alternative approach
→ Time/Space Complexity
→ Interview follow-ups
→ Hinglish 30-sec answer
```
