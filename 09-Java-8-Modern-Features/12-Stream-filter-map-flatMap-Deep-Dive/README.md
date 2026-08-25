# 9.12 — Stream `filter()` / `map()` / `flatMap()` Deep Dive

## 🎯 Interview Goal

Understand the three most important Stream transformation patterns:

```text
filter()  → select elements
map()     → transform each element
flatMap() → transform + flatten nested structures
```

---

# 1. `filter()` — Deep Dive ⭐⭐⭐⭐⭐

`filter()` keeps only elements that satisfy a `Predicate`.

```java
List<Integer> numbers = Arrays.asList(10, 15, 20, 25, 30);

List<Integer> evenNumbers = numbers.stream()
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());

System.out.println(evenNumbers);
```

Output:

```text
[10, 20, 30]
```

### Mental model

```text
Input → condition → keep/discard
```

`filter()` does **not** transform the element type.

Conceptually:

```java
Stream<T> → Stream<T>
```

---

# 2. Multiple `filter()` Conditions

```java
List<Integer> result = numbers.stream()
        .filter(n -> n > 10)
        .filter(n -> n % 2 == 0)
        .collect(Collectors.toList());
```

Equivalent condition:

```java
List<Integer> result = numbers.stream()
        .filter(n -> n > 10 && n % 2 == 0)
        .collect(Collectors.toList());
```

### Interview point

Multiple filters can make individual business conditions easier to read. Performance and readability should both be considered rather than blindly preferring one form.

---

# 3. `filter()` With Objects ⭐⭐⭐⭐⭐

```java
class Employee {
    private String name;
    private String department;
    private int salary;

    Employee(String name, String department, int salary) {
        this.name = name;
        this.department = department;
        this.salary = salary;
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
}
```

Filter IT employees:

```java
List<Employee> itEmployees = employees.stream()
        .filter(e -> "IT".equals(e.getDepartment()))
        .collect(Collectors.toList());
```

Filter high-paid IT employees:

```java
List<Employee> result = employees.stream()
        .filter(e -> "IT".equals(e.getDepartment()))
        .filter(e -> e.getSalary() > 100000)
        .collect(Collectors.toList());
```

---

# 4. `map()` — Deep Dive ⭐⭐⭐⭐⭐

`map()` transforms every element using a `Function`.

```java
List<String> names = Arrays.asList("java", "spring", "kafka");

List<String> result = names.stream()
        .map(String::toUpperCase)
        .collect(Collectors.toList());
```

Output:

```text
[JAVA, SPRING, KAFKA]
```

Conceptually:

```text
T → R
```

Examples:

```java
Stream<String> → Stream<Integer>
Stream<Employee> → Stream<String>
Stream<Integer> → Stream<Integer>
```

---

# 5. `map()` Can Change the Type ⭐⭐⭐⭐⭐

```java
List<String> names = Arrays.asList("Nirbhay", "Rahul", "Priya");

List<Integer> lengths = names.stream()
        .map(String::length)
        .collect(Collectors.toList());
```

Output:

```text
[7, 5, 5]
```

Here:

```text
String → Integer
```

---

# 6. Employee → Employee Name ⭐⭐⭐⭐⭐

```java
List<String> employeeNames = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Flow:

```text
Employee
   ↓
getName()
   ↓
String
```

This is one of the most common interview patterns.

---

# 7. `filter()` + `map()` Together ⭐⭐⭐⭐⭐

Very common interview question:

> Find names of IT employees whose salary is greater than 10 LPA.

```java
List<String> result = employees.stream()
        .filter(e -> "IT".equals(e.getDepartment()))
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Pipeline:

```text
Employees
    ↓
filter department
    ↓
filter salary
    ↓
map name
    ↓
collect List<String>
```

---

# 8. `map()` vs `filter()` 🔥

| `filter()` | `map()` |
|---|---|
| Selects elements | Transforms elements |
| Takes Predicate | Takes Function |
| Result type usually remains T | Result type can become R |
| Can reduce number of elements | Normally produces one result per input |
| Example: salary > 10L | Example: Employee → name |

### 2-second answer

> `filter()` decides **which elements survive**; `map()` decides **what each surviving element becomes**.

---

# 9. Why `flatMap()` Is Needed ⭐⭐⭐⭐⭐

Suppose we have:

```java
List<List<Integer>> numbers = Arrays.asList(
        Arrays.asList(1, 2),
        Arrays.asList(3, 4),
        Arrays.asList(5, 6)
);
```

Using `map()`:

```java
List<Stream<Integer>> result = numbers.stream()
        .map(List::stream)
        .collect(Collectors.toList());
```

You get a nested structure conceptually:

```text
Stream<Stream<Integer>>
```

But we want:

```text
Stream<Integer>
```

That is where `flatMap()` comes in.

---

# 10. `flatMap()` — Core Concept ⭐⭐⭐⭐⭐

```java
List<Integer> result = numbers.stream()
        .flatMap(List::stream)
        .collect(Collectors.toList());
```

Output:

```text
[1, 2, 3, 4, 5, 6]
```

Mental model:

```text
map()

List<List<Integer>>
       ↓
List<Stream<Integer>>
       ↓
Nested structure ❌
```

```text
flatMap()

List<List<Integer>>
       ↓
Stream<List<Integer>>
       ↓
flatten
       ↓
Stream<Integer>
       ↓
[1,2,3,4,5,6] ✅
```

---

# 11. `map()` vs `flatMap()` — Most Important Interview Question 🔥🔥🔥

Input:

```java
List<List<String>> data = Arrays.asList(
        Arrays.asList("Java", "Spring"),
        Arrays.asList("Kafka", "Docker")
);
```

### Using `map()`

```java
List<Stream<String>> result = data.stream()
        .map(List::stream)
        .collect(Collectors.toList());
```

Conceptual result:

```text
List<Stream<String>>
```

### Using `flatMap()`

```java
List<String> result = data.stream()
        .flatMap(List::stream)
        .collect(Collectors.toList());
```

Result:

```text
[Java, Spring, Kafka, Docker]
```

### Interview answer

> `map()` performs one-to-one transformation and can produce nested structures when each input maps to multiple values. `flatMap()` maps each input to a Stream and then flattens those nested Streams into a single Stream.

---

# 12. `flatMap()` With Employees and Skills ⭐⭐⭐⭐⭐

Suppose:

```java
class Employee {
    private String name;
    private List<String> skills;

    public String getName() {
        return name;
    }

    public List<String> getSkills() {
        return skills;
    }
}
```

Get all skills:

```java
List<String> allSkills = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .distinct()
        .collect(Collectors.toList());
```

Flow:

```text
Employee 1 → [Java, Spring]
Employee 2 → [Kafka, Java]
Employee 3 → [Docker]

             ↓ flatMap

[Java, Spring, Kafka, Java, Docker]
             ↓ distinct
[Java, Spring, Kafka, Docker]
```

---

# 13. `flatMap()` With Optional ⭐⭐⭐⭐⭐

This is a very important connection with the previous Optional chapter.

Suppose:

```java
Optional<Address> getAddress()
```

If we use `map()`:

```java
Optional<Optional<Address>> result =
        employee.getOptionalAddress()
                .map(address -> address);
```

Nested Optional can appear when the mapper itself returns Optional.

With `flatMap()`:

```java
Optional<Address> result =
        employee.getOptionalAddress()
                .flatMap(address -> address.getOptionalCity());
```

### Remember

```text
Stream flatMap() → flatten nested Streams
Optional flatMap() → flatten nested Optional
```

The underlying idea is similar: avoid unnecessary nesting.

---

# 14. `flatMap()` With Strings 🔥

Question:

> Given a list of sentences, return all unique words.

Input:

```java
List<String> sentences = Arrays.asList(
        "Java is powerful",
        "Java is popular",
        "Spring is powerful"
);
```

Solution:

```java
List<String> words = sentences.stream()
        .flatMap(sentence -> Arrays.stream(sentence.split("\\s+")))
        .distinct()
        .collect(Collectors.toList());
```

Flow:

```text
Sentence
   ↓
split()
   ↓
Stream<String>
   ↓
flatMap()
   ↓
Single Stream<String>
   ↓
distinct()
```

---

# 15. `filter()` Inside `flatMap()`

Get only Java-related skills:

```java
List<String> javaSkills = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .filter(skill -> skill.toLowerCase().contains("java"))
        .distinct()
        .collect(Collectors.toList());
```

Pipeline:

```text
Employee
 ↓
flatMap skills
 ↓
filter Java skills
 ↓
distinct
 ↓
collect
```

---

# 16. Nested List — Three-Level Example ⭐⭐⭐⭐⭐

```java
List<List<List<Integer>>> data = Arrays.asList(
        Arrays.asList(
                Arrays.asList(1, 2),
                Arrays.asList(3, 4)
        ),
        Arrays.asList(
                Arrays.asList(5, 6)
        )
);
```

Flatten one level:

```java
Stream<List<Integer>> level1 = data.stream()
        .flatMap(List::stream);
```

Flatten two levels:

```java
List<Integer> result = data.stream()
        .flatMap(List::stream)
        .flatMap(List::stream)
        .collect(Collectors.toList());
```

Result:

```text
[1, 2, 3, 4, 5, 6]
```

---

# 17. `flatMap()` Does Not Mean “Remove All Nesting”

Important interview nuance:

`flatMap()` flattens **one Stream level per invocation**.

For three nested levels, you may need multiple `flatMap()` calls.

```java
stream
    .flatMap(...)
    .flatMap(...);
```

---

# 18. Null Handling With `flatMap()` ⚠️

Avoid returning `null` from a `flatMap()` mapper.

Bad:

```java
.flatMap(e -> e.getSkills() == null
        ? null
        : e.getSkills().stream())
```

Prefer an empty Stream:

```java
.flatMap(e -> e.getSkills() == null
        ? Stream.empty()
        : e.getSkills().stream())
```

Or, when appropriate in modern Java, use a null-safe design at the object boundary.

---

# 19. `filter()` + `map()` + `flatMap()` Complete Example ⭐⭐⭐⭐⭐

Requirement:

> Find unique Java-related skills of IT employees earning more than 10 LPA.

```java
List<String> result = employees.stream()
        .filter(e -> "IT".equals(e.getDepartment()))
        .filter(e -> e.getSalary() > 100000)
        .flatMap(e -> e.getSkills().stream())
        .filter(skill -> skill.toLowerCase().contains("java"))
        .map(String::toUpperCase)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
```

This is a very realistic 5-year Java interview pipeline.

---

# 20. Order Matters 🔥

Compare:

```java
stream
    .filter(condition)
    .map(transformation)
```

with:

```java
stream
    .map(transformation)
    .filter(condition)
```

They are not always equivalent.

Example:

```java
numbers.stream()
        .filter(n -> n > 10)
        .map(n -> n * 2);
```

Filtering first can avoid transforming values that will later be discarded.

### Interview point

> Put selective, inexpensive filters early when that improves the pipeline, while preserving correctness and readability.

---

# 21. Stream Processing Is Element-Oriented

Example:

```java
Stream.of(1, 2, 3)
        .filter(n -> {
            System.out.println("filter " + n);
            return n > 1;
        })
        .map(n -> {
            System.out.println("map " + n);
            return n * 10;
        })
        .forEach(System.out::println);
```

Conceptually:

```text
1 → filter → discarded
2 → filter → map → result
3 → filter → map → result
```

The Stream pipeline does not simply execute all filters first and all maps afterward as separate collection passes.

---

# 22. `map()` With Null Values ⚠️

`map()` itself can produce null values if the mapper returns null:

```java
List<String> result = names.stream()
        .map(name -> null)
        .collect(Collectors.toList());
```

That is legal, but usually undesirable.

If null values must be removed:

```java
names.stream()
        .map(this::convert)
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
```

---

# 23. `flatMap()` With Empty Streams

An empty Stream contributes no element.

```java
List<Integer> result = Stream.of(
        Arrays.asList(1, 2),
        Collections.emptyList(),
        Arrays.asList(3, 4)
)
.flatMap(List::stream)
.collect(Collectors.toList());
```

Result:

```text
[1, 2, 3, 4]
```

This makes `flatMap()` useful for optional/empty nested data.

---

# 24. Common Interview Trap — `map()` Returning Collection

Suppose:

```java
List<Employee> employees;
```

and:

```java
Employee::getSkills
```

returns `List<String>`.

Then:

```java
List<List<String>> result = employees.stream()
        .map(Employee::getSkills)
        .collect(Collectors.toList());
```

This is correct if you want each employee's skill list.

But if you want one combined skill list:

```java
List<String> result = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .collect(Collectors.toList());
```

### Decision rule

```text
Need nested result?  → map()
Need flattened result? → flatMap()
```

---

# 25. Complete Runnable Practice Code

```java
import java.util.*;
import java.util.stream.Collectors;

public class FilterMapFlatMapDemo {

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
            return name + " - " + department + " - " + salary;
        }
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000,
                        Arrays.asList("Java", "Spring Boot", "Kafka")),
                new Employee(2, "Rahul", "IT", 90000,
                        Arrays.asList("Java", "Docker")),
                new Employee(3, "Priya", "HR", 120000,
                        Arrays.asList("Java", "Excel")),
                new Employee(4, "Amit", "IT", 130000,
                        Arrays.asList("Java", "AWS", "Spring Boot"))
        );

        // 1. filter
        List<Employee> itEmployees = employees.stream()
                .filter(e -> "IT".equals(e.getDepartment()))
                .collect(Collectors.toList());

        // 2. map: Employee -> String
        List<String> employeeNames = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.toList());

        // 3. map: String -> Integer
        List<Integer> nameLengths = employeeNames.stream()
                .map(String::length)
                .collect(Collectors.toList());

        // 4. filter + map
        List<String> highPaidNames = employees.stream()
                .filter(e -> e.getSalary() > 100000)
                .map(Employee::getName)
                .collect(Collectors.toList());

        // 5. map creates nested result
        List<List<String>> employeeSkills = employees.stream()
                .map(Employee::getSkills)
                .collect(Collectors.toList());

        // 6. flatMap creates one combined list
        List<String> allSkills = employees.stream()
                .flatMap(e -> e.getSkills().stream())
                .collect(Collectors.toList());

        // 7. unique skills
        List<String> uniqueSkills = employees.stream()
                .flatMap(e -> e.getSkills().stream())
                .distinct()
                .sorted()
                .collect(Collectors.toList());

        // 8. realistic interview pipeline
        List<String> javaSkillsOfEligibleITEmployees = employees.stream()
                .filter(e -> "IT".equals(e.getDepartment()))
                .filter(e -> e.getSalary() > 100000)
                .flatMap(e -> e.getSkills().stream())
                .filter(skill -> skill.toLowerCase().contains("java"))
                .map(String::toUpperCase)
                .distinct()
                .sorted()
                .collect(Collectors.toList());

        // 9. sentences -> words
        List<String> sentences = Arrays.asList(
                "Java is powerful",
                "Java is popular",
                "Spring is powerful"
        );

        List<String> uniqueWords = sentences.stream()
                .flatMap(sentence -> Arrays.stream(sentence.split("\\s+")))
                .distinct()
                .sorted()
                .collect(Collectors.toList());

        // 10. nested list flattening
        List<List<Integer>> nestedNumbers = Arrays.asList(
                Arrays.asList(1, 2),
                Arrays.asList(3, 4),
                Arrays.asList(5, 6)
        );

        List<Integer> flattenedNumbers = nestedNumbers.stream()
                .flatMap(List::stream)
                .collect(Collectors.toList());

        System.out.println("IT employees: " + itEmployees);
        System.out.println("Employee names: " + employeeNames);
        System.out.println("Name lengths: " + nameLengths);
        System.out.println("High paid names: " + highPaidNames);
        System.out.println("Employee skills: " + employeeSkills);
        System.out.println("All skills: " + allSkills);
        System.out.println("Unique skills: " + uniqueSkills);
        System.out.println("Eligible Java skills: " +
                javaSkillsOfEligibleITEmployees);
        System.out.println("Unique words: " + uniqueWords);
        System.out.println("Flattened numbers: " + flattenedNumbers);
    }
}
```

---

# 26. Output Prediction 🔥

What is the result?

```java
List<List<Integer>> data = Arrays.asList(
        Arrays.asList(1, 2),
        Arrays.asList(3, 4)
);

List<Integer> result = data.stream()
        .flatMap(List::stream)
        .filter(n -> n % 2 == 0)
        .map(n -> n * 10)
        .collect(Collectors.toList());
```

Answer:

```text
[20, 40]
```

---

# 27. Interview Trap — `map()` vs `flatMap()`

```java
List<List<String>> data = Arrays.asList(
        Arrays.asList("A", "B"),
        Arrays.asList("C", "D")
);
```

### Question 1

```java
data.stream()
    .map(List::stream)
```

Type?

```text
Stream<Stream<String>>
```

### Question 2

```java
data.stream()
    .flatMap(List::stream)
```

Type?

```text
Stream<String>
```

This type reasoning is extremely important in interviews.

---

# 28. 20 Interview Questions 🔥🔥🔥

1. What does `filter()` do?
2. What functional interface does `filter()` accept?
3. What does `map()` do?
4. What functional interface does `map()` accept?
5. Can `map()` change the element type?
6. Why is `map()` not enough for nested collections?
7. What problem does `flatMap()` solve?
8. What is the difference between `map()` and `flatMap()`?
9. What is the type of `list.stream().map(List::stream)`?
10. What is the type after `flatMap(List::stream)`?
11. Can `flatMap()` flatten multiple nesting levels in one call?
12. How does `flatMap()` work with empty collections?
13. How do you get all employee skills using Streams?
14. How do you remove duplicate skills?
15. How do you find Java skills of IT employees?
16. Why should selective filters often be placed early?
17. Can `map()` return null?
18. Should a `flatMap()` mapper return null?
19. How is Optional `flatMap()` conceptually similar to Stream `flatMap()`?
20. Explain a `filter → flatMap → filter → map → distinct → sorted → collect` pipeline in 2 minutes.

---

# 29. Coding Challenges 🔥🔥🔥

### Challenge 1
Filter all even numbers.

### Challenge 2
Convert every name to uppercase.

### Challenge 3
Convert employee objects to employee names.

### Challenge 4
Find employees earning more than 10 LPA and return their names.

### Challenge 5
Given `List<List<Integer>>`, flatten it into `List<Integer>`.

### Challenge 6
Flatten nested lists and return only even numbers.

### Challenge 7
Given employees with `List<String> skills`, return all unique skills.

### Challenge 8
Return all unique skills sorted alphabetically.

### Challenge 9
Return only skills containing `java`, case-insensitively.

### Challenge 10 ⭐⭐⭐⭐⭐
Build this pipeline:

```text
Employees
 ↓
IT department
 ↓
salary > 10 LPA
 ↓
flatMap skills
 ↓
filter Java-related skills
 ↓
uppercase
 ↓
distinct
 ↓
sorted
 ↓
List<String>
```

### Challenge 11
Given sentences, return all unique words.

### Challenge 12
Given `List<List<List<Integer>>>`, flatten all levels.

### Challenge 13
Explain why `map(List::stream)` does not produce `List<String>`.

### Challenge 14
Demonstrate the difference between `map()` and `flatMap()` using employees and skills.

### Challenge 15 — 5-Year Interview Level ⭐⭐⭐⭐⭐
Given a nested domain object, design a Stream pipeline that safely handles empty child collections and returns a sorted, distinct final result without introducing nested collections.

---

# 30. Common Mistakes ❌

### ❌ Mistake 1
Using `map()` when a flattened result is required.

### ❌ Mistake 2
Thinking `flatMap()` simply means “remove all nesting.”

It flattens the Stream level represented by that operation.

### ❌ Mistake 3
Returning `null` from `flatMap()`.

Use `Stream.empty()` where appropriate.

### ❌ Mistake 4
Doing expensive `map()` work before a selective filter when the transformation is unnecessary for rejected elements.

### ❌ Mistake 5
Using `map()` to collect child collections when the requirement is one combined collection.

---

# 31. Final Revision Sheet 🧠

```text
filter()
────────────────────────
Predicate<T>
Selects elements
T → T
```

```text
map()
────────────────────────
Function<T,R>
Transforms elements
T → R
```

```text
flatMap()
────────────────────────
Function<T, Stream<R>>
Transforms + flattens
T → Stream<R> → R
```

### Golden Rule

```text
Need to SELECT?       → filter()
Need to TRANSFORM?    → map()
Need to FLATTEN?      → flatMap()
```

### Most important interview example

```java
employees.stream()
        .filter(e -> "IT".equals(e.getDepartment()))
        .filter(e -> e.getSalary() > 100000)
        .flatMap(e -> e.getSkills().stream())
        .filter(skill -> skill.toLowerCase().contains("java"))
        .map(String::toUpperCase)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
```

---

# 32. 2-Minute Interview Script 🎤

> “In the Stream API, `filter`, `map`, and `flatMap` solve different problems. `filter` accepts a Predicate and selects elements that satisfy a condition. `map` accepts a Function and transforms each element, so it can also change the element type, for example Employee to String. `flatMap` is used when each input produces a nested Stream or collection and I need one flattened Stream. For example, if every Employee has a list of skills, `map(Employee::getSkills)` gives a nested `List<List<String>>`, whereas `flatMap(e -> e.getSkills().stream())` produces one Stream of skills. In real applications I commonly combine them: filter the relevant employees, flatMap their child collections, apply another filter, map to the required output, then use distinct, sorted and collect.”

---

# 🧪 Complete Practice Code

urlGitHub — 9.12 Stream filter/map/flatMap Deep Dive Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/12-Stream-filter-map-flatMap-Deep-Dive

---

## Navigation

[← 9.11 — Java 8 Stream API Fundamentals](../11-Stream-API-Fundamentals/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.12 — Stream `filter()` / `map()` / `flatMap()` Deep Dive → ✅ Completed**

**Next → 9.13 — Stream `sorted()` / `distinct()` / `limit()` / `skip()`**