# 9.22 — Stream API Best Practices & Anti-Patterns

## 🎯 Interview Goal

Master how to write readable, correct, maintainable and production-ready Stream API code—and recognize when a stream is the wrong tool.

> **Core rule:** Use Streams to express data transformations clearly. Do not use Streams merely to make code shorter.

---

# 1. Prefer Readability Over Clever One-Liners ⭐⭐⭐⭐⭐

### ❌ Hard to read

```java
Map<String, Double> result = employees.stream().filter(e -> e.getSalary() > 100000).collect(Collectors.groupingBy(Employee::getDepartment, Collectors.averagingInt(Employee::getSalary)));
```

### ✅ Readable

```java
Map<String, Double> averageSalaryByDepartment = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

Interview line:

> “Streams should improve readability, not turn business logic into compressed one-liners.”

---

# 2. Don't Use Streams for Everything ❌

### Simple loop can be better

```java
for (Employee employee : employees) {
    if (employee.getSalary() > 100000) {
        System.out.println(employee.getName());
    }
}
```

A stream is not automatically better:

```java
employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .forEach(e -> System.out.println(e.getName()));
```

For simple imperative logic, the loop can sometimes be easier to debug and understand.

---

# 3. Avoid `peek()` for Business Logic ⚠️🔥

### ❌ Anti-pattern

```java
employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .peek(e -> sendEmail(e))
        .collect(Collectors.toList());
```

`peek()` is primarily intended for observing/debugging a pipeline, not for required business side effects.

### Better

```java
List<Employee> eligible = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .collect(Collectors.toList());

eligible.forEach(this::sendEmail);
```

---

# 4. Remember: Streams Are Lazy 🔥🔥🔥

This does nothing by itself:

```java
employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName);
```

There is no terminal operation.

Add one:

```java
List<String> names = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Interview question:

> Why wasn't the `filter()` executed?

Answer:

> “Intermediate stream operations are lazy. They execute when a terminal operation consumes the pipeline.”

---

# 5. Avoid Reusing a Stream ❌🔥

### ❌

```java
Stream<Employee> stream = employees.stream();

long count = stream.count();

List<Employee> result = stream.collect(Collectors.toList());
```

This throws:

```text
IllegalStateException: stream has already been operated upon or closed
```

### ✅

Create a new stream:

```java
long count = employees.stream().count();

List<Employee> result = employees.stream()
        .collect(Collectors.toList());
```

---

# 6. Prefer Primitive Streams for Numeric Work ⭐⭐⭐⭐

### Less ideal

```java
int total = employees.stream()
        .map(Employee::getSalary)
        .reduce(0, Integer::sum);
```

### Better for numeric aggregation

```java
int total = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

Useful APIs:

```text
mapToInt()
mapToLong()
mapToDouble()
IntStream
LongStream
DoubleStream
```

This can avoid unnecessary boxing and makes numeric intent explicit.

---

# 7. Filter Early When It Reduces Work ⭐⭐⭐⭐⭐

### Better

```java
List<String> names = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Instead of transforming every element first:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .filter(name -> /* cannot use salary now */ false)
        .collect(Collectors.toList());
```

General principle:

```text
Reduce data as early as correctness allows.
```

But don't reorder operations if doing so changes semantics.

---

# 8. Avoid Repeated Traversals When One Pipeline Is Enough

### ❌

```java
long count = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .count();

int total = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .mapToInt(Employee::getSalary)
        .sum();
```

This traverses the source twice.

Sometimes this is perfectly acceptable for readability. If one pass is materially important, consider a suitable collector or a dedicated aggregation object.

Do not optimize this blindly—measure first.

---

# 9. Don't Create Unnecessary Intermediate Collections ❌

### Unnecessary

```java
List<Employee> filtered = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .collect(Collectors.toList());

List<String> names = filtered.stream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

### Better

```java
List<String> names = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Unless the intermediate collection has a real purpose, keep the pipeline together.

---

# 10. `map()` vs `flatMap()` 🔥

### One-to-one transformation

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

### One-to-many flattening

```java
List<String> skills = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .distinct()
        .collect(Collectors.toList());
```

Mental model:

```text
map()
Employee → String

flatMap()
Employee → Stream<String>
then flatten
```

---

# 11. Don't Use `map()` When You Need Flattening ❌

### ❌

```java
List<List<String>> skills = employees.stream()
        .map(Employee::getSkills)
        .collect(Collectors.toList());
```

### ✅

```java
List<String> skills = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .collect(Collectors.toList());
```

---

# 12. Handle Nulls Explicitly ⚠️

### ❌

```java
employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .collect(Collectors.toList());
```

If `getSkills()` can return null, this can throw `NullPointerException`.

### Better domain design

Prefer returning an empty collection:

```java
private List<String> getSkills() {
    return skills == null ? Collections.emptyList() : skills;
}
```

Then:

```java
employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .collect(Collectors.toList());
```

---

# 13. Don't Overuse `Optional` Inside Streams

Avoid turning every value into nested `Optional` pipelines just to avoid a simple null check.

Use `Optional` where it clearly communicates absence, especially at API boundaries.

---

# 14. `orElse()` vs `orElseGet()` Performance Trap 🔥

```java
String value = optional.orElse(expensiveFallback());
```

The fallback expression is evaluated before `orElse()` receives it.

With lazy fallback:

```java
String value = optional.orElseGet(this::expensiveFallback);
```

Interview line:

> “`orElse()` evaluates its argument eagerly, while `orElseGet()` obtains the fallback lazily from a supplier.”

---

# 15. Avoid `get()` on Optional Without a Reason ❌

### ❌

```java
String name = employees.stream()
        .filter(e -> e.getId() == 10)
        .map(Employee::getName)
        .findFirst()
        .get();
```

If no element exists, `NoSuchElementException` is thrown.

Better:

```java
String name = employees.stream()
        .filter(e -> e.getId() == 10)
        .map(Employee::getName)
        .findFirst()
        .orElse("Unknown");
```

Or:

```java
String name = employees.stream()
        .filter(e -> e.getId() == 10)
        .map(Employee::getName)
        .findFirst()
        .orElseThrow(() -> new IllegalArgumentException("Employee not found"));
```

---

# 16. Prefer `toList()` Carefully

Modern Java provides:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .toList();
```

But remember:

```text
Stream.toList() → unmodifiable list
Collectors.toList() → mutability is not guaranteed by the Collector contract
```

If you explicitly need a mutable `ArrayList`:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toCollection(ArrayList::new));
```

---

# 17. Avoid `Collectors.toMap()` Duplicate-Key Surprise 🔥

### ❌

```java
Map<String, Employee> map = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity()
        ));
```

If multiple employees have the same department, this can throw:

```text
IllegalStateException: Duplicate key
```

### ✅ Merge function

```java
Map<String, Employee> map = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                (e1, e2) -> e1
        ));
```

---

# 18. Don't Use `distinct()` as a Substitute for Correct Equality

```java
employees.stream()
        .distinct()
        .collect(Collectors.toList());
```

`distinct()` relies on object equality semantics.

If `Employee` does not correctly implement `equals()` and `hashCode()`, logical duplicates may not be removed as expected.

---

# 19. Be Careful With `sorted()` 🔥

```java
employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary))
        .collect(Collectors.toList());
```

Sorting is generally more expensive than simple filtering/mapping and is stateful.

Do not sort if you only need a maximum/minimum:

### Instead of

```java
Employee highest = employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .findFirst()
        .orElse(null);
```

Prefer:

```java
Employee highest = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary))
        .orElse(null);
```

This avoids sorting the entire dataset.

---

# 20. `limit()` Before Expensive Work When Semantically Valid ⭐⭐⭐⭐

```java
List<String> names = employees.stream()
        .limit(10)
        .map(this::expensiveTransformation)
        .collect(Collectors.toList());
```

If you only need ten elements, there is no reason to transform every element first.

But always preserve semantics.

---

# 21. Don't Abuse `peek()` to Debug Everything

For temporary debugging:

```java
employees.stream()
        .peek(e -> System.out.println("Before filter: " + e))
        .filter(e -> e.getSalary() > 100000)
        .peek(e -> System.out.println("After filter: " + e))
        .collect(Collectors.toList());
```

Useful during debugging, but remove noisy logging from production pipelines unless it is intentionally designed.

---

# 22. Avoid Side Effects in `map()` ❌

### ❌

```java
employees.stream()
        .map(e -> {
            auditService.log(e);
            return e.getName();
        })
        .collect(Collectors.toList());
```

`map()` should conceptually transform values.

If auditing is a required side effect, make that side effect explicit rather than hiding it inside a transformation.

---

# 23. `forEach()` Is a Terminal Operation, Not a Transformation

### ❌

```java
List<String> result = new ArrayList<>();

employees.stream()
        .forEach(e -> result.add(e.getName()));
```

### ✅

```java
List<String> result = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Use:

```text
map()      → transform
filter()   → select
collect()  → accumulate
forEach()  → perform terminal side effect
```

---

# 24. Avoid Nested Streams That Become Hard to Read

### Hard to maintain

```java
employees.stream()
        .filter(e -> e.getProjects().stream()
                .filter(p -> p.isActive())
                .anyMatch(Project::isCritical))
        .map(Employee::getName)
        .collect(Collectors.toList());
```

This can still be valid, but if business rules grow, extract meaningful methods:

```java
List<String> names = employees.stream()
        .filter(this::hasActiveCriticalProject)
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Readable stream + readable domain method is usually better.

---

# 25. Don't Hide Business Rules Inside Giant Lambdas ❌

### ❌

```java
employees.stream()
        .filter(e -> e.getSalary() > 100000 &&
                e.getExperience() >= 5 &&
                e.getDepartment() != null &&
                !e.getDepartment().equals("HR") &&
                e.isActive())
        .collect(Collectors.toList());
```

### ✅

```java
employees.stream()
        .filter(this::isEligibleForPromotion)
        .collect(Collectors.toList());
```

```java
private boolean isEligibleForPromotion(Employee e) {
    return e.getSalary() > 100000
            && e.getExperience() >= 5
            && e.getDepartment() != null
            && !e.getDepartment().equals("HR")
            && e.isActive();
}
```

---

# 26. Choose the Correct Terminal Operation ⭐⭐⭐⭐⭐

```text
count()       → number of elements
anyMatch()    → at least one
allMatch()    → every element
noneMatch()   → no elements match
findFirst()   → first matching element
findAny()     → any matching element
min()/max()   → extremum
reduce()      → custom reduction
collect()     → mutable/structured aggregation
```

Do not collect everything into a list just to ask a yes/no question.

### ❌

```java
boolean exists = employees.stream()
        .filter(e -> e.getSalary() > 200000)
        .collect(Collectors.toList())
        .size() > 0;
```

### ✅

```java
boolean exists = employees.stream()
        .anyMatch(e -> e.getSalary() > 200000);
```

---

# 27. Short-Circuiting Can Save Work 🔥

```java
boolean exists = employees.stream()
        .anyMatch(e -> e.getSalary() > 200000);
```

Once a match is found, processing can stop.

Other short-circuiting operations include:

```text
findFirst()
findAny()
anyMatch()
allMatch()
noneMatch()
limit()
```

---

# 28. Don't Call Expensive Methods Repeatedly

### ❌

```java
employees.stream()
        .filter(e -> expensiveCheck(e))
        .filter(e -> expensiveCheck(e))
        .collect(Collectors.toList());
```

If the same expensive calculation is needed repeatedly, consider restructuring the pipeline or caching the result where appropriate.

Do not introduce caching blindly—consider memory and correctness.

---

# 29. Avoid Accidental N+1 Database Calls 🔥🔥🔥

### ❌

```java
employees.stream()
        .map(e -> employeeRepository.findProjects(e.getId()))
        .flatMap(Collection::stream)
        .collect(Collectors.toList());
```

If `findProjects()` executes a database query, this may execute one query per employee.

Better design could involve:

```text
fetch required data in bulk
JOIN / batch query
then stream in memory
```

Streams do not eliminate database round trips.

---

# 30. Don't Use Parallel Streams as a Database Optimization ❌🔥

### ❌

```java
employees.parallelStream()
        .map(e -> employeeRepository.findProjects(e.getId()))
        .collect(Collectors.toList());
```

This can increase database concurrency and overwhelm the database.

Parallelism must respect downstream resource limits.

---

# 31. Stream Pipelines Should Usually Be Non-Interfering

Avoid modifying the source collection while traversing it:

```java
employees.stream()
        .forEach(employees::remove);
```

This can cause `ConcurrentModificationException` and violates the intended non-interfering stream usage model.

---

# 32. Don't Assume Stream Order Is Free

Ordered stream:

```java
employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Parallel pipelines may need coordination to preserve encounter order.

If order genuinely does not matter, an unordered parallel pipeline can sometimes remove unnecessary constraints:

```java
employees.parallelStream()
        .unordered()
        .filter(Employee::isActive)
        .count();
```

Only do this when semantics permit it.

---

# 33. Don't Use `parallelStream()` Without Evidence ⭐⭐⭐⭐⭐

Bad reasoning:

```text
10 million records
→ parallelStream() must be faster
```

Correct reasoning:

```text
data size
+ CPU cost
+ source splitting
+ ordering
+ synchronization
+ downstream resources
+ common-pool usage
+ benchmark
```

---

# 34. Complete Runnable Practice Code 💻

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class StreamBestPracticesDemo {

    static class Employee {
        private final int id;
        private final String name;
        private final String department;
        private final int salary;
        private final int experience;
        private final boolean active;
        private final List<String> skills;

        Employee(int id, String name, String department,
                 int salary, int experience, boolean active,
                 List<String> skills) {
            this.id = id;
            this.name = name;
            this.department = department;
            this.salary = salary;
            this.experience = experience;
            this.active = active;
            this.skills = skills == null
                    ? Collections.emptyList()
                    : skills;
        }

        public int getId() { return id; }
        public String getName() { return name; }
        public String getDepartment() { return department; }
        public int getSalary() { return salary; }
        public int getExperience() { return experience; }
        public boolean isActive() { return active; }
        public List<String> getSkills() { return skills; }

        @Override
        public String toString() {
            return id + " - " + name + " - " + department + " - " + salary;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Employee)) return false;
            Employee employee = (Employee) o;
            return id == employee.id;
        }

        @Override
        public int hashCode() {
            return Integer.hashCode(id);
        }
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000, 7, true,
                        Arrays.asList("Java", "Spring", "SQL")),
                new Employee(2, "Rahul", "IT", 120000, 5, true,
                        Arrays.asList("Java", "AWS")),
                new Employee(3, "Amit", "HR", 90000, 4, true,
                        Arrays.asList("Excel", "Communication")),
                new Employee(4, "Priya", "Finance", 130000, 6, true,
                        Arrays.asList("SQL", "Excel")),
                new Employee(5, "Sneha", "IT", 110000, 3, false,
                        Arrays.asList("Java", "Docker"))
        );

        // 1. Readable filter + map + collect
        List<String> highPaidNames = employees.stream()
                .filter(e -> e.getSalary() > 100000)
                .map(Employee::getName)
                .collect(Collectors.toList());

        // 2. Primitive stream for numeric aggregation
        int totalSalary = employees.stream()
                .mapToInt(Employee::getSalary)
                .sum();

        // 3. anyMatch instead of collect + size
        boolean highSalaryExists = employees.stream()
                .anyMatch(e -> e.getSalary() > 200000);

        // 4. max instead of sorting the complete collection
        Employee highestPaid = employees.stream()
                .max(Comparator.comparingInt(Employee::getSalary))
                .orElse(null);

        // 5. flatMap for one-to-many transformation
        List<String> skills = employees.stream()
                .flatMap(e -> e.getSkills().stream())
                .distinct()
                .sorted()
                .collect(Collectors.toList());

        // 6. Safe toMap with duplicate-key merge function
        Map<String, Employee> employeeByDepartment = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getDepartment,
                        Function.identity(),
                        (e1, e2) -> e1
                ));

        // 7. groupingBy for one-to-many relationship
        Map<String, List<Employee>> byDepartment = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment));

        // 8. Short-circuiting
        Optional<Employee> firstHighPaid = employees.stream()
                .filter(e -> e.getSalary() > 140000)
                .findFirst();

        // 9. Explicit business rule method
        List<Employee> promotionEligible = employees.stream()
                .filter(StreamBestPracticesDemo::isEligibleForPromotion)
                .collect(Collectors.toList());

        // 10. Order is irrelevant: unordered parallel pipeline
        long activeCount = employees.parallelStream()
                .unordered()
                .filter(Employee::isActive)
                .count();

        // 11. Demonstrate stream laziness
        employees.stream()
                .filter(e -> {
                    System.out.println("Filtering: " + e.getName());
                    return e.getSalary() > 100000;
                });

        System.out.println("Nothing above is printed from the lazy pipeline.");

        employees.stream()
                .filter(e -> {
                    System.out.println("Executed filter: " + e.getName());
                    return e.getSalary() > 100000;
                })
                .count();

        // 12. forEach as explicit terminal side effect
        highPaidNames.forEach(System.out::println);

        System.out.println("High paid names: " + highPaidNames);
        System.out.println("Total salary: " + totalSalary);
        System.out.println("High salary exists: " + highSalaryExists);
        System.out.println("Highest paid: " + highestPaid);
        System.out.println("Skills: " + skills);
        System.out.println("By department: " + byDepartment);
        System.out.println("First high paid: " + firstHighPaid);
        System.out.println("Promotion eligible: " + promotionEligible);
        System.out.println("Active count: " + activeCount);
    }

    private static boolean isEligibleForPromotion(Employee e) {
        return e.getSalary() > 100000
                && e.getExperience() >= 5
                && e.getDepartment() != null
                && !e.getDepartment().equals("HR")
                && e.isActive();
    }
}
```

---

# 35. Anti-Pattern → Correct Pattern 🔥🔥🔥

| Anti-pattern | Better approach |
|---|---|
| Giant stream one-liner | Break pipeline / extract method |
| `peek()` for business logic | Explicit side effect after collection |
| Reuse a stream | Create a new stream |
| `map()` for flattening | `flatMap()` |
| `reduce()` for simple sum | `mapToInt().sum()` |
| Sort then `findFirst()` for max | `max()` |
| Collect just to check existence | `anyMatch()` |
| `forEach()` + external list | `collect()` |
| `toMap()` without merge when duplicates possible | Add merge function / use grouping |
| `parallelStream()` everywhere | Benchmark + workload analysis |
| Stream for complex imperative workflow | Consider a loop/service method |
| Stream causing N+1 queries | Bulk-fetch data |

---

# 36. 25 Interview Questions 🎯

1. What are the best practices for Stream API?
2. Why shouldn't streams be used for everything?
3. Why is `peek()` not recommended for business logic?
4. Explain stream laziness.
5. Can a stream be reused?
6. What happens when a stream is reused?
7. `map()` vs `flatMap()`?
8. Why use primitive streams?
9. `mapToInt()` vs `map()`?
10. Why filter early?
11. When should you use `max()` instead of `sorted()`?
12. What are short-circuiting operations?
13. `anyMatch()` vs collecting and checking size?
14. Why is side-effecting `forEach()` often an anti-pattern?
15. Why is shared mutable state dangerous?
16. What happens if `toMap()` receives duplicate keys?
17. How does the merge function of `toMap()` work?
18. How does `distinct()` depend on `equals()` and `hashCode()`?
19. What does non-interference mean?
20. Why can parallel streams be dangerous with blocking I/O?
21. Can streams prevent N+1 database queries?
22. Why should business logic be extracted from giant lambdas?
23. `orElse()` vs `orElseGet()`?
24. `Stream.toList()` vs `Collectors.toList()`?
25. When would you choose a loop over a stream in production code?

---

# 37. 25 Coding Challenges 💻

1. Rewrite a nested loop using streams.
2. Rewrite an over-complicated stream using a loop.
3. Find the maximum salary without sorting.
4. Find the second-highest salary.
5. Find duplicate employee IDs.
6. Remove duplicates using correct equality semantics.
7. Flatten employee skills using `flatMap()`.
8. Handle null skill lists safely.
9. Group employees by department.
10. Convert employees to a map with duplicate-key handling.
11. Find whether any employee earns above a threshold.
12. Check whether all employees are active.
13. Find the first employee matching a condition.
14. Compare `findFirst()` and `findAny()` using a parallel stream.
15. Calculate salary using `mapToInt()`.
16. Rewrite a `reduce()` sum using primitive streams.
17. Demonstrate stream laziness.
18. Demonstrate stream reuse failure.
19. Demonstrate `peek()` for debugging only.
20. Detect an N+1-style repository call in stream code.
21. Replace `parallelStream()` with a safer design for I/O.
22. Rewrite side-effecting `forEach()` into `collect()`.
23. Build a department salary summary.
24. Create a readable stream for a complex promotion rule.
25. **5-year interview challenge:** Review a production stream pipeline and identify at least 10 correctness, performance, readability, and maintainability issues.

---

# 38. 5-Year Interview Scenario 🔥🔥🔥

**Question:** “I have this stream pipeline in production. Should I optimize it?”

Strong answer:

> “First I would establish the actual problem using metrics or profiling. Then I would inspect the source size, pipeline operations, allocation/boxing, database or network calls, ordering requirements, side effects, and whether the pipeline is executed sequentially or in parallel. I would avoid speculative optimization. If a change is justified, I would benchmark representative workloads and verify correctness with tests.”

---

# 39. Golden Rules 🧠

```text
Readable > clever

Transform → map()
Filter    → filter()
Flatten   → flatMap()
Aggregate → collect()/reduce()
Numeric   → mapToInt()/mapToLong()
Exists    → anyMatch()
All       → allMatch()
None      → noneMatch()
First     → findFirst()
Any       → findAny()
Min/Max   → min()/max()
Debug     → peek() temporarily
Business side effect → explicit code
```

And:

```text
Don't reuse streams
Don't mutate the source
Don't hide business rules in giant lambdas
Don't blindly use parallelStream()
Don't create accidental N+1 queries
Don't collect when a short-circuit operation is enough
Don't sort when max()/min() solves the problem
```

---

# 40. 2-Minute Interview Script 🎤

> “My main Stream API principle is readability first. I use streams for clear transformations, filtering and aggregation, but I don't force every piece of logic into a stream. I avoid shared mutable state and business side effects inside `map()` or `peek()`, and I remember that intermediate operations are lazy and streams cannot be reused. For numeric operations I prefer primitive streams such as `mapToInt()`. I use short-circuiting operations like `anyMatch()` or `findFirst()` instead of collecting data unnecessarily, and I use `max()` or `min()` instead of sorting when I only need an extremum. I also watch for duplicate keys with `toMap()`, correct equality for `distinct()`, null handling in `flatMap()`, and accidental N+1 database calls. For parallel streams, I don't assume they are faster; I consider workload, CPU versus I/O, ordering, shared state, downstream resource limits and benchmark results. Finally, if a stream becomes harder to understand than a loop, I prefer the simpler design.”

---

# 🧪 Complete Practice Code

[GitHub — 9.22 Stream API Best Practices & Anti-Patterns Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/22-Stream-API-Best-Practices-and-Anti-Patterns)

---

## Navigation

[← 9.21 — Advanced Stream Performance & Parallel Streams](../21-Advanced-Stream-Performance-and-Parallel-Streams-Deep-Dive/README.md)

**Current → 9.22 — Stream API Best Practices & Anti-Patterns → ✅ Completed**

**Next → 9.23 — Stream API Interview Scenarios & Final Revision**