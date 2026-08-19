# Q02 — Print All Unique Departments

## 🎯 Problem

Given a list of `Employee` objects, print all **unique department names**.

Example:

```text
Input:
Amit    → IT
Priya   → HR
Rahul   → IT
Neha    → Finance
Karan   → HR
```

Expected output:

```text
IT
HR
Finance
```

`IT` and `HR` repeat in the employee data, but output mein har department sirf ek baar aana chahiye.

---

# 1. Approach

Requirement ko break karo:

```text
Employee List
     ↓
   stream()
     ↓
map(Employee → Department)
     ↓
  distinct()
     ↓
   forEach()
```

Main Java 8 mein:

```java
map() + distinct()
```

use karunga.

---

# 2. Java 8 Solution

```java
employeeList.stream()
        .map(Employee::getDepartment)
        .distinct()
        .forEach(System.out::println);
```

Ye directly unique department names print karega.

---

# 3. Line-by-Line Explanation

### Step 1 — `stream()`

```java
employeeList.stream()
```

List ko Stream mein convert karta hai.

```text
List<Employee>
      ↓
Stream<Employee>
```

---

### Step 2 — `map()`

```java
.map(Employee::getDepartment)
```

Employee object ko department name mein convert karta hai.

Before:

```text
Employee
Employee
Employee
```

After:

```text
IT
HR
IT
Finance
HR
```

Yahan stream ka type roughly:

```text
Stream<Employee>
      ↓ map()
Stream<String>
```

---

### Step 3 — `distinct()`

```java
.distinct()
```

Duplicate department values remove karta hai.

Before:

```text
IT
HR
IT
Finance
HR
```

After:

```text
IT
HR
Finance
```

---

### Step 4 — `forEach()`

```java
.forEach(System.out::println);
```

Har unique department ko print karta hai.

---

# 4. Complete Flow

```text
             List<Employee>
                    ↓
                 stream()
                    ↓
          map(Employee::getDepartment)
                    ↓
        IT, HR, IT, Finance, HR
                    ↓
                 distinct()
                    ↓
             IT, HR, Finance
                    ↓
                 forEach()
                    ↓
                 Print
```

---

# 5. Dry Run

Suppose input:

```text
[
  Amit   → IT,
  Priya  → HR,
  Rahul  → IT,
  Neha   → Finance,
  Karan  → HR
]
```

### After `map()`

```text
[IT, HR, IT, Finance, HR]
```

### After `distinct()`

```text
[IT, HR, Finance]
```

### Output

```text
IT
HR
Finance
```

---

# 6. Why `map()` Is Required?

A common interview question:

> Why can't you directly call `distinct()` on the employee stream?

Because:

```java
employeeList.stream()
        .distinct()
```

would remove duplicate **Employee objects**, not duplicate department names.

Requirement is:

```text
Employee
   ↓
Department
   ↓
Unique Department
```

Therefore first:

```java
map(Employee::getDepartment)
```

then:

```java
distinct()
```

---

# 7. `map()` vs `filter()`

### `map()`

Transforms each element.

```java
.map(Employee::getDepartment)
```

Meaning:

```text
Employee → String
```

### `filter()`

Selects/removes elements based on a condition.

```java
.filter(e -> e.getAge() > 25)
```

Meaning:

```text
Employee → Employee
```

Memory trick:

> **map = transform, filter = select.**

---

# 8. Why `distinct()`?

`distinct()` is a **stateful intermediate operation**.

It keeps track of previously seen values so duplicate values can be removed.

Conceptually:

```text
IT       → keep
HR       → keep
IT       → duplicate → remove
Finance  → keep
HR       → duplicate → remove
```

---

# 9. How Does `distinct()` Know Values Are Duplicate?

For object values, distinctness is based on equality semantics, effectively using `equals()` and `hashCode()` to identify duplicates.

For `String` department names:

```java
"IT".equals("IT")
```

is `true`, so repeated `"IT"` values are treated as duplicates.

This is also why understanding `equals()` and `hashCode()` is important for Java interviews.

---

# 10. Important Interview Point — Order

For an ordered sequential stream, `distinct()` preserves the encounter order of the first occurrence.

Example:

```text
IT
HR
IT
Finance
HR
```

Result:

```text
IT
HR
Finance
```

So the first occurrence is retained.

Do not confuse this with sorting.

`distinct()` does **not** alphabetically sort the departments.

---

# 11. `distinct()` vs `sorted()`

### `distinct()`

Removes duplicates:

```java
.distinct()
```

### `sorted()`

Sorts elements:

```java
.sorted()
```

If you need unique departments in alphabetical order:

```java
employeeList.stream()
        .map(Employee::getDepartment)
        .distinct()
        .sorted()
        .forEach(System.out::println);
```

Flow:

```text
map
 ↓
distinct
 ↓
sorted
 ↓
print
```

---

# 12. If We Need a `Set` Instead of Printing

The question says **print**, so `forEach()` is enough.

But if the requirement is:

> Return unique departments.

then collect into a Set:

```java
Set<String> departments = employeeList.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toSet());
```

A `Set` naturally represents unique values.

---

# 13. `distinct()` vs `toSet()`

### `distinct()`

```java
stream
    .distinct()
    .forEach(...);
```

It removes duplicates **inside the stream pipeline**.

### `Collectors.toSet()`

```java
stream
    .collect(Collectors.toSet());
```

It collects the final values into a Set.

Use based on what the next operation needs.

---

# 14. If Order Matters

If you need unique departments while preserving insertion/encounter order in the resulting collection:

```java
Set<String> departments = employeeList.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toCollection(LinkedHashSet::new));
```

This is useful when the returned collection's order matters.

Do not blindly assume `HashSet` preserves order.

---

# 15. Alternative — `HashSet`

Without Streams:

```java
Set<String> departments = new HashSet<>();

for (Employee employee : employeeList) {
    departments.add(employee.getDepartment());
}

for (String department : departments) {
    System.out.println(department);
}
```

This is completely valid.

But because the question is specifically about Java 8 features, the Stream solution is more appropriate for demonstrating `map()` and `distinct()`.

---

# 16. Alternative — `TreeSet` for Sorted Output

If the requirement says:

> Print unique departments in sorted order.

Use:

```java
Set<String> departments = employeeList.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toCollection(TreeSet::new));
```

Then:

```text
Finance
HR
IT
```

instead of encounter order.

---

# 17. Null Department Handling

If a department can be null, decide the business requirement first.

If `null` should be ignored:

```java
employeeList.stream()
        .map(Employee::getDepartment)
        .filter(Objects::nonNull)
        .distinct()
        .forEach(System.out::println);
```

Flow:

```text
Employee
 ↓
Department
 ↓
Remove null
 ↓
Remove duplicates
 ↓
Print
```

If `null` should be displayed as `UNKNOWN`:

```java
employeeList.stream()
        .map(e -> e.getDepartment() == null
                ? "UNKNOWN"
                : e.getDepartment())
        .distinct()
        .forEach(System.out::println);
```

---

# 18. Case Sensitivity

Consider:

```text
IT
it
It
```

`distinct()` treats these as different Strings because String equality is case-sensitive.

If the requirement is case-insensitive:

```java
employeeList.stream()
        .map(Employee::getDepartment)
        .filter(Objects::nonNull)
        .map(String::toLowerCase)
        .distinct()
        .forEach(System.out::println);
```

But this changes the displayed value to lowercase.

If you need case-insensitive uniqueness while preserving the original spelling, you need a different approach, such as tracking normalized keys separately.

---

# 19. Time Complexity

Let:

```text
N = number of employees
```

`map()` processes each employee:

```text
O(N)
```

`distinct()` needs to track seen values. With hash-based tracking, expected processing is approximately:

```text
O(N)
```

Therefore overall expected time:

```text
O(N)
```

If there are `K` distinct departments:

```text
Space = O(K)
```

because `distinct()` needs to remember values it has already seen.

---

# 20. Why Isn't Space O(1)?

Because `distinct()` must remember previously encountered department values.

Example:

```text
10,000 different departments
```

Then it may need to track many values.

So:

```text
Space = O(K)
```

where `K` is the number of distinct departments.

---

# 21. Is `distinct()` Always O(N)?

For interview-level analysis, say:

> **Expected O(N) time with hash-based tracking.**

There are implementation and stream characteristics that can affect exact behavior, especially for parallel streams, but O(N) expected is the appropriate answer for the normal sequential case.

---

# 22. Interview Question — Why Not Use `groupingBy()`?

You could do:

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

But that solves a different requirement:

```text
Department → Employee count
```

Our requirement is only:

```text
Unique department names
```

So:

```java
map() + distinct()
```

is simpler and more precise.

---

# 23. Interview Question — Can You Combine Q1 and Q2?

Yes.

For example, if interviewer asks:

> Give department-wise employee counts and unique departments.

The grouping result already contains the unique department keys:

```java
Map<String, Long> result = employeeList.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.counting()
        ));
```

Then:

```java
result.keySet().forEach(System.out::println);
```

But don't use this if you only need unique departments; it does extra work by counting.

---

# 24. Interview Question — `distinct()` vs `Set`

### `distinct()`

Best when you want to continue a Stream pipeline:

```java
stream
    .map(...)
    .distinct()
    .sorted()
    .forEach(...);
```

### `Set`

Best when the final data structure itself should represent uniqueness:

```java
Set<String> departments = ...;
```

Simple rule:

> **Pipeline mein uniqueness → `distinct()`; result collection mein uniqueness → `Set`.**

---

# 25. 2-Minute Interview Answer

> **"I would first transform the employee stream into a stream of department names using `map(Employee::getDepartment)`. Since departments can repeat across employees, I would use `distinct()` to remove duplicate department names and then print the result with `forEach()`. The pipeline is `stream → map → distinct → forEach`. The expected time complexity is O(N), where N is the number of employees, and the additional space is O(K), where K is the number of distinct departments, because `distinct()` needs to track previously seen values. If the requirement is to return the departments instead of printing them, I could collect them into a Set."**

---

# 26. 30-Second Hinglish Answer

> **"Mujhe employee objects se sirf department names chahiye, isliye pehle `map(Employee::getDepartment)` se Employee ko Department String mein convert karunga. Phir `distinct()` duplicate departments remove karega aur `forEach()` unique departments print karega. Pattern simple hai: `map → distinct → print`. Expected time O(N) aur extra space O(K) hai, jahan K distinct departments hain. Agar return karna ho instead of print, to `Set` use kar sakte hain."**

---

# 27. 🧠 Memory Trick

```text
Employee
   ↓
map()
   ↓
Department
   ↓
distinct()
   ↓
Unique Department
   ↓
forEach()
   ↓
Print
```

### One-line rule

> **"Object se field nikalni hai → `map()`, duplicate hataane hain → `distinct()`."**

---

# 28. Common Interview Mistakes

### ❌ Mistake 1

```java
employeeList.stream().distinct()
```

This does not directly solve unique department names.

### ❌ Mistake 2

Using `filter()` instead of `map()` to extract department.

### ❌ Mistake 3

Assuming `distinct()` sorts the result.

It doesn't.

### ❌ Mistake 4

Claiming `distinct()` needs O(1) extra space.

It generally needs to track seen values.

### ❌ Mistake 5

Using `HashSet` when encounter order is a requirement without explaining ordering.

---

# 29. Follow-Up Questions

1. What does `map()` do in this solution?
2. What does `distinct()` use to identify duplicates?
3. Difference between `map()` and `filter()`?
4. Difference between `distinct()` and `Set`?
5. How do you sort the unique departments?
6. How do you preserve insertion order?
7. What happens if department is `null`?
8. How do you make uniqueness case-insensitive?
9. Why not use `groupingBy()` here?
10. What is the time and space complexity?
11. Is `distinct()` stateful or stateless?
12. How would this behave with a parallel stream?

---

# 🔗 Parent Chapter

[Java 8 Coding Questions — Employee Management System](../README.md)

Previous: [Q01 — Count Male and Female Employees](../Q01-Count-Male-and-Female-Employees/README.md)

## Status

✅ **Q02 Completed**

Next: **Q03 — Find average age of male and female employees**
