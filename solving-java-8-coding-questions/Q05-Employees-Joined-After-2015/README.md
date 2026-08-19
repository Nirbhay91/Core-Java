# Q05 — Find Employees Who Joined After 2015

## 🎯 Problem

Given a list of `Employee` objects, find/print all employees who **joined after the year 2015**.

Example:

```text
Amit   → 2014
Rahul  → 2016
Priya  → 2018
Neha   → 2015
Karan  → 2020
```

Expected output:

```text
Rahul  → 2016
Priya  → 2018
Karan  → 2020
```

### Important wording

> **After 2015** means strictly greater than 2015.

Therefore:

```text
2015 → ❌
2016 → ✅
```

---

# 1. Approach

Requirement ko break karo:

```text
Employee List
      ↓
    stream()
      ↓
   filter()
      ↓
joiningYear > 2015
      ↓
matching employees
      ↓
   forEach()
```

Java 8 mein main:

```java
filter() + condition
```

use karunga.

---

# 2. Java 8 Solution — Year as `int`

Agar Employee mein joining year directly `int` hai:

```java
employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .forEach(System.out::println);
```

---

# 3. If We Need a List Instead of Printing

Question agar ho:

> Return all employees who joined after 2015.

Use:

```java
List<Employee> result = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .collect(Collectors.toList());
```

Java 16+ mein `toList()` bhi available hai:

```java
List<Employee> result = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .toList();
```

But Java 8 interview mein:

```java
Collectors.toList()
```

use karna appropriate hai.

---

# 4. Line-by-Line Explanation

### Step 1 — `stream()`

```java
employeeList.stream()
```

Converts:

```text
List<Employee>
      ↓
Stream<Employee>
```

---

### Step 2 — `filter()`

```java
.filter(e -> e.getJoiningYear() > 2015)
```

`filter()` sirf un employees ko aage pass karega jo condition satisfy karte hain.

Conceptually:

```text
2014 → false → reject
2015 → false → reject
2016 → true  → keep
2018 → true  → keep
2020 → true  → keep
```

---

### Step 3 — `forEach()`

```java
.forEach(System.out::println);
```

Remaining employees ko print karta hai.

---

# 5. Complete Flow

```text
                 List<Employee>
                       ↓
                    stream()
                       ↓
                     filter
                       ↓
             joiningYear > 2015?
                       ↓
          ┌────────────┴────────────┐
          ↓                         ↓
        false                       true
          ↓                         ↓
       discard                    keep
                                    ↓
                                forEach()
                                    ↓
                                  print
```

---

# 6. Dry Run

Input:

```text
Amit   → 2014
Rahul  → 2016
Priya  → 2018
Neha   → 2015
Karan  → 2020
```

### Amit

```text
2014 > 2015 → false
```

Reject.

### Rahul

```text
2016 > 2015 → true
```

Keep.

### Priya

```text
2018 > 2015 → true
```

Keep.

### Neha

```text
2015 > 2015 → false
```

Reject.

### Karan

```text
2020 > 2015 → true
```

Keep.

Final:

```text
Rahul
Priya
Karan
```

---

# 7. Why `filter()`?

Interview mein simple answer:

> **"`filter()` is used when I want to keep only the elements that satisfy a condition."**

Here:

```java
joiningYear > 2015
```

is our predicate.

Pattern:

```text
Input elements
      ↓
   filter()
      ↓
Condition true → keep
Condition false → discard
```

---

# 8. `filter()` vs `map()`

Very common follow-up.

### `filter()`

Selects elements.

```java
.filter(e -> e.getJoiningYear() > 2015)
```

Type remains:

```text
Employee → Employee
```

### `map()`

Transforms elements.

```java
.map(Employee::getName)
```

Type changes:

```text
Employee → String
```

Memory:

> **filter = select, map = transform.**

---

# 9. What Is a Predicate?

This lambda:

```java
e -> e.getJoiningYear() > 2015
```

is a predicate-like condition used by `filter()`.

Conceptually:

```java
Predicate<Employee>
```

A `Predicate<T>` represents a boolean-valued function:

```java
boolean test(T value)
```

For this problem:

```text
Employee → true/false
```

---

# 10. Complete Interview-Ready Code

```java
public static List<Employee> findEmployeesJoinedAfter2015(
        List<Employee> employeeList) {

    if (employeeList == null || employeeList.isEmpty()) {
        return Collections.emptyList();
    }

    return employeeList.stream()
            .filter(Objects::nonNull)
            .filter(e -> e.getJoiningYear() > 2015)
            .collect(Collectors.toList());
}
```

This version explicitly handles:

```text
null list
empty list
null Employee elements
```

---

# 11. If Joining Date Is `LocalDate`

In a real project, storing only the year may not be enough.

Suppose:

```java
private LocalDate joiningDate;
```

Then the requirement:

> Joined after 2015

could mean:

> Joined after **2015-12-31**.

Use:

```java
LocalDate cutoff = LocalDate.of(2015, 12, 31);

List<Employee> result = employeeList.stream()
        .filter(e -> e.getJoiningDate() != null)
        .filter(e -> e.getJoiningDate().isAfter(cutoff))
        .collect(Collectors.toList());
```

This is more precise than extracting just the year.

---

# 12. `Year` API Alternative

If the business requirement is explicitly year-based and the domain stores a `Year`:

```java
private Year joiningYear;
```

Then:

```java
Year cutoff = Year.of(2015);

List<Employee> result = employeeList.stream()
        .filter(e -> e.getJoiningYear() != null)
        .filter(e -> e.getJoiningYear().isAfter(cutoff))
        .collect(Collectors.toList());
```

This communicates the domain meaning better than a raw `int`.

---

# 13. Important Date Boundary Clarification

"After 2015" can be interpreted as:

```text
Year > 2015
```

or as a date boundary:

```text
joiningDate > 2015-12-31
```

Usually these are equivalent for date-only values, but the exact requirement should be clarified.

Do not accidentally implement:

```java
>= 2015
```

because that includes employees who joined in 2015.

---

# 14. If Requirement Is "2015 or Later"

Then condition changes to:

```java
.filter(e -> e.getJoiningYear() >= 2015)
```

Difference:

```text
After 2015
→ > 2015

2015 or later
→ >= 2015
```

This tiny operator difference is a common interview trap.

---

# 15. Filter + Map Example

Suppose interviewer says:

> Print only the names of employees who joined after 2015.

Then combine:

```java
employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .map(Employee::getName)
        .forEach(System.out::println);
```

Flow:

```text
Employee
   ↓
filter(year > 2015)
   ↓
Employee
   ↓
map(name)
   ↓
String
   ↓
print
```

This is a very common Java 8 pattern.

---

# 16. Filter + Sort

If interviewer asks:

> Find employees who joined after 2015 and sort them by joining year.

Use:

```java
List<Employee> result = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .sorted(Comparator.comparingInt(Employee::getJoiningYear))
        .collect(Collectors.toList());
```

Now the pipeline is:

```text
filter
 ↓
sort
 ↓
collect
```

---

# 17. Filter + Count

If interviewer asks:

> Count employees who joined after 2015.

Use:

```java
long count = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .count();
```

This is another important pattern:

```text
condition-based count
→ filter() + count()
```

---

# 18. Filter + Find First

If interviewer asks:

> Find the first employee who joined after 2015.

Use:

```java
Optional<Employee> result = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .findFirst();
```

Because there may be no matching employee:

```java
Optional<Employee>
```

is returned.

---

# 19. `findFirst()` vs `findAny()`

### `findFirst()`

Returns the first element according to encounter order for an ordered stream.

### `findAny()`

Returns some matching element and can provide more flexibility for parallel processing.

For a normal sequential interview problem:

```java
findFirst()
```

is usually what you want when "first" is part of the requirement.

---

# 20. Filter + Partition

If interviewer asks:

> Separate employees into joined after 2015 and not joined after 2015.

Use `partitioningBy()`:

```java
Map<Boolean, List<Employee>> result = employeeList.stream()
        .collect(Collectors.partitioningBy(
                e -> e.getJoiningYear() > 2015
        ));
```

Result conceptually:

```text
true  → employees joined after 2015
false → everyone else
```

This connects directly with Q1/Q3's collector concepts.

---

# 21. Filter + Grouping

If interviewer asks:

> Find department-wise employees who joined after 2015.

Use:

```java
Map<String, List<Employee>> result = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Flow:

```text
Employee List
     ↓
filter(year > 2015)
     ↓
matching employees
     ↓
groupingBy(department)
     ↓
Department → Employees
```

---

# 22. Time Complexity

Let:

```text
N = number of employees
```

Each employee is checked once:

```text
Time = O(N)
```

If we only filter and collect, additional result space is:

```text
O(M)
```

where `M` is the number of employees who satisfy the condition.

If the question only asks to print the matching employees, there is no separate output collection required by the algorithm itself.

---

# 23. Does `filter()` Modify the Original List?

No.

Streams do not modify the source collection simply because we call `filter()`.

```java
employeeList.stream()
        .filter(...)
```

creates a pipeline over the source.

If you collect the result:

```java
.collect(Collectors.toList())
```

you get a separate result list.

---

# 24. Interview Question — Is `filter()` Intermediate or Terminal?

`filter()` is an **intermediate operation**.

It returns another Stream:

```java
Stream<Employee>
```

Examples of intermediate operations:

```text
filter()
map()
distinct()
sorted()
```

Terminal operations:

```text
forEach()
collect()
count()
max()
min()
findFirst()
```

Memory:

> **Intermediate = pipeline continue hota hai.**
> **Terminal = pipeline end hota hai.**

---

# 25. Interview Question — Is `filter()` Lazy?

Yes.

This alone does not immediately process employees:

```java
Stream<Employee> stream = employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015);
```

Processing happens when a terminal operation is invoked:

```java
.collect(Collectors.toList());
```

or:

```java
.forEach(...);
```

So:

```text
Source
 ↓
filter pipeline created
 ↓
No terminal operation
 ↓
No actual traversal yet
```

---

# 26. Multiple Filters

You can chain conditions:

```java
employeeList.stream()
        .filter(e -> e.getJoiningYear() > 2015)
        .filter(e -> e.getSalary() > 50000)
        .collect(Collectors.toList());
```

Equivalent conceptually to:

```java
.filter(e ->
        e.getJoiningYear() > 2015 &&
        e.getSalary() > 50000
)
```

For readability, separate filters can sometimes make the business rules clearer.

---

# 27. `filter()` vs `removeIf()`

`filter()` is used in a Stream pipeline and does not mutate the source list by itself.

`removeIf()` is a collection operation that **mutates** the collection:

```java
employeeList.removeIf(e -> e.getJoiningYear() <= 2015);
```

This changes the original list.

For interview questions asking to find/return matching elements:

```java
filter()
```

is generally the appropriate Stream solution.

---

# 28. Real-World Generalization

The same pattern works for many business requirements:

### Salary

```java
.filter(e -> e.getSalary() > 100000)
```

### Age

```java
.filter(e -> e.getAge() > 30)
```

### Department

```java
.filter(e -> "IT".equals(e.getDepartment()))
```

### Active employees

```java
.filter(Employee::isActive)
```

Generic rule:

```text
Need only matching records
        ↓
      filter()
```

---

# 29. Q1 → Q5 Java 8 Pattern

Now notice how the questions are building your Stream fundamentals:

### Q1 — Gender-wise count

```text
groupingBy() + counting()
```

### Q2 — Unique departments

```text
map() + distinct()
```

### Q3 — Average age by gender

```text
groupingBy() + averagingInt()
```

### Q4 — Highest-paid employee

```text
max() + Comparator
```

### Q5 — Joined after 2015

```text
filter() + condition
```

These are reusable patterns, not five unrelated programs.

---

# 30. 2-Minute Interview Answer

> **"I would use the Stream `filter()` operation because the requirement is to select employees based on a condition. If the employee stores joining year as an integer, the condition is `e.getJoiningYear() > 2015`. If I need to return the matching employees, I would collect the filtered stream using `Collectors.toList()`. This is O(N) time because every employee is checked once. The result collection requires O(M) space, where M is the number of matching employees. I would also clarify the boundary: 'after 2015' means strictly greater than 2015, so employees who joined in 2015 are excluded. If the real model stores a `LocalDate`, I would use a date boundary and `isAfter()` rather than relying only on the year."**

---

# 31. 30-Second Hinglish Answer

> **"Employees ko joining year ke basis par filter karna hai, so main `filter()` use karunga: `e.getJoiningYear() > 2015`. Yahan `>` important hai because 'after 2015' ka matlab 2015 include nahi hoga. Agar result return karna hai to `collect(Collectors.toList())` karunga. Time complexity O(N) hai, aur result ke liye O(M) space lagega jahan M matching employees hain. Agar actual field `LocalDate` hai to cutoff date ke saath `isAfter()` use karna better hai."**

---

# 32. 🧠 Memory Trick

```text
Requirement:
"Only employees satisfying a condition"
              ↓
          filter()
              ↓
       condition true
              ↓
            keep
```

### One-line rule

> **"Sirf matching records chahiye → `filter()`."**

---

# 33. Common Interview Mistakes

### ❌ Mistake 1

Using:

```java
>= 2015
```

when the requirement says **after 2015**.

### ❌ Mistake 2

Using `map()` instead of `filter()`.

### ❌ Mistake 3

Confusing `filter()` with `removeIf()`.

`removeIf()` mutates the original collection.

### ❌ Mistake 4

Forgetting that `filter()` is lazy until a terminal operation runs.

### ❌ Mistake 5

Using only the year when the real business requirement is date/time based.

### ❌ Mistake 6

Ignoring null joining dates.

---

# 34. Follow-Up Questions

1. What is `filter()` in Java 8?
2. Is `filter()` intermediate or terminal?
3. Is `filter()` lazy?
4. Difference between `filter()` and `map()`?
5. Difference between `filter()` and `removeIf()`?
6. How do you count employees who joined after 2015?
7. How do you print only their names?
8. How do you sort them by joining year?
9. How do you group them by department?
10. How do you partition employees into before/after 2015?
11. What if joining date is `LocalDate` instead of an integer year?
12. What is the time complexity?
13. What is the difference between `>` and `>=` in this requirement?
14. How would you handle null employees or null dates?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

Previous: [Q04 — Highest-Paid Employee](../Q04-Highest-Paid-Employee/README.md)

## Status

✅ **Q05 Completed**

Next: **Q06 — Count employees in each department**
