# 9.23 — Stream API Interview Scenarios & Final Revision

## 🎯 Interview Goal

This is the final Stream API revision chapter for a 5-year Java developer interview. The goal is not just to remember syntax, but to choose the correct operation, explain trade-offs, write production-quality code, and solve realistic interview problems.

---

# 1. Stream API Mental Model ⭐⭐⭐⭐⭐

```text
Source
  ↓
Stream
  ↓
Intermediate operations
(filter / map / flatMap / sorted / distinct / peek)
  ↓
Terminal operation
(collect / reduce / count / find / match / forEach)
```

### Key rule

> Intermediate operations are lazy. The pipeline runs only when a terminal operation consumes it.

---

# 2. Operation Selection Cheat Sheet 🧠

| Requirement | Preferred operation |
|---|---|
| Select elements | `filter()` |
| Transform each element | `map()` |
| Flatten nested data | `flatMap()` |
| Remove duplicates | `distinct()` |
| Sort | `sorted()` |
| Take first N | `limit()` |
| Skip first N | `skip()` |
| Sum numbers | `mapToInt().sum()` |
| Custom aggregation | `reduce()` |
| Build collection | `collect()` |
| Group data | `groupingBy()` |
| Split into true/false | `partitioningBy()` |
| Create map | `toMap()` |
| Join strings | `joining()` |
| Check existence | `anyMatch()` |
| Check all | `allMatch()` |
| Check none | `noneMatch()` |
| First element | `findFirst()` |
| Any element | `findAny()` |
| Minimum | `min()` |
| Maximum | `max()` |
| Count | `count()` |
| Terminal side effect | `forEach()` |

---

# 3. Scenario — Find Highest Paid Employee 🔥

### ❌ Unnecessary sorting

```java
Employee highest = employees.stream()
        .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
        .findFirst()
        .orElse(null);
```

### ✅ Better

```java
Employee highest = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary))
        .orElse(null);
```

### Interview line

> “If I only need the maximum, I don't need to sort the entire stream.”

---

# 4. Scenario — Second Highest Salary ⭐⭐⭐⭐⭐

```java
Optional<Integer> secondHighest = employees.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();
```

### Follow-up

What if duplicate salaries should count as separate employees?

Then remove `distinct()`:

```java
Optional<Integer> secondHighestEmployeeSalary = employees.stream()
        .map(Employee::getSalary)
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();
```

---

# 5. Scenario — Group Employees by Department

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

### Average salary by department

```java
Map<String, Double> averageSalary = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingInt(Employee::getSalary)
        ));
```

### Highest paid employee by department

```java
Map<String, Optional<Employee>> highestByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

---

# 6. Scenario — Convert List to Map With Duplicate Keys 🔥

### Risky

```java
Map<String, Employee> map = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity()
        ));
```

If duplicate departments exist, this can throw `IllegalStateException`.

### Safe merge

```java
Map<String, Employee> map = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                (e1, e2) -> e1
        ));
```

### Interview question

> What does the third argument of `toMap()` do?

Answer:

> “It is the merge function used when multiple input elements produce the same key.”

---

# 7. Scenario — Flatten Employee Skills

```java
List<String> skills = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .distinct()
        .sorted()
        .collect(Collectors.toList());
```

Mental model:

```text
Employee → List<String>
          ↓ flatMap
       String
```

---

# 8. Scenario — Find Duplicate Values

```java
Set<Integer> seen = new HashSet<>();

Set<Integer> duplicates = employees.stream()
        .map(Employee::getId)
        .filter(id -> !seen.add(id))
        .collect(Collectors.toSet());
```

### Interview caveat

This uses mutable state inside the stream and is best treated as an interview/controlled sequential example. It is not a good pattern for a parallel stream.

A frequency-map approach is often clearer:

```java
Map<Integer, Long> frequency = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getId,
                Collectors.counting()
        ));

Set<Integer> duplicates = frequency.entrySet().stream()
        .filter(entry -> entry.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toSet());
```

---

# 9. Scenario — Partition Employees

```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
        .collect(Collectors.partitioningBy(Employee::isActive));
```

Use `partitioningBy()` when the classification is specifically boolean.

---

# 10. Scenario — Find Employees Above Salary Threshold

### ❌ Unnecessary collection

```java
boolean exists = employees.stream()
        .filter(e -> e.getSalary() > 200000)
        .collect(Collectors.toList())
        .size() > 0;
```

### ✅ Short-circuit

```java
boolean exists = employees.stream()
        .anyMatch(e -> e.getSalary() > 200000);
```

---

# 11. Scenario — All Employees Active

```java
boolean allActive = employees.stream()
        .allMatch(Employee::isActive);
```

### No inactive employees

```java
boolean noneInactive = employees.stream()
        .noneMatch(e -> !e.isActive());
```

---

# 12. Scenario — `findFirst()` vs `findAny()` 🔥

Sequential stream:

```java
Optional<Employee> first = employees.stream()
        .filter(Employee::isActive)
        .findFirst();
```

Parallel stream:

```java
Optional<Employee> any = employees.parallelStream()
        .filter(Employee::isActive)
        .findAny();
```

### Interview answer

> “`findFirst()` respects encounter order when one exists. `findAny()` is allowed to return any matching element and can provide more flexibility for parallel execution.”

---

# 13. Scenario — Calculate Total Salary

### Preferred numeric form

```java
int totalSalary = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

### Why?

```text
map()       → Stream<Integer>
mapToInt()  → IntStream
```

Primitive streams can avoid boxing overhead and clearly express numeric aggregation.

---

# 14. Scenario — Reduce Interview Question 🔥

```java
int total = employees.stream()
        .map(Employee::getSalary)
        .reduce(0, Integer::sum);
```

### Better for simple integer sum

```java
int total = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

### Interview line

> “I use `reduce()` when I need a custom reduction. For standard numeric sums, primitive stream operations are clearer.”

---

# 15. Scenario — Why Is `reduce()` Tricky in Parallel Streams? ⭐⭐⭐⭐⭐

The reduction function should be associative for reliable parallel reduction.

Good:

```java
int sum = numbers.parallelStream()
        .reduce(0, Integer::sum);
```

Dangerous reasoning:

```text
“Any reduction operation will work the same sequentially and in parallel.”
```

Not true. The accumulator/combiner semantics and identity must be compatible with parallel execution.

---

# 16. Scenario — `map()` vs `flatMap()`

```java
List<List<String>> nested = employees.stream()
        .map(Employee::getSkills)
        .collect(Collectors.toList());
```

versus:

```java
List<String> flat = employees.stream()
        .flatMap(e -> e.getSkills().stream())
        .collect(Collectors.toList());
```

### Interview line

> “`map()` preserves one output per input, while `flatMap()` transforms each input into a stream and then flattens those streams.”

---

# 17. Scenario — Stream Laziness 🔥

```java
employees.stream()
        .filter(e -> {
            System.out.println("filter: " + e.getName());
            return e.getSalary() > 100000;
        })
        .map(Employee::getName);
```

No terminal operation → no processing.

Add:

```java
.count();
```

Now the pipeline executes.

---

# 18. Scenario — Stream Cannot Be Reused

```java
Stream<Employee> stream = employees.stream();

long count = stream.count();

// IllegalStateException
stream.collect(Collectors.toList());
```

Correct:

```java
long count = employees.stream().count();
List<Employee> list = employees.stream().collect(Collectors.toList());
```

---

# 19. Scenario — `peek()` Anti-Pattern

### ❌

```java
employees.stream()
        .peek(e -> sendEmail(e))
        .filter(Employee::isActive)
        .collect(Collectors.toList());
```

### Better

```java
List<Employee> activeEmployees = employees.stream()
        .filter(Employee::isActive)
        .collect(Collectors.toList());

activeEmployees.forEach(this::sendEmail);
```

`peek()` is useful for observation/debugging, not for required business side effects.

---

# 20. Scenario — N+1 Database Problem 🔥🔥🔥

### Suspicious

```java
List<Project> projects = employees.stream()
        .map(e -> employeeRepository.findProjects(e.getId()))
        .flatMap(Collection::stream)
        .collect(Collectors.toList());
```

If every repository call hits the database, this can generate one query per employee.

### Production thinking

```text
Stream API does not remove database round trips.

Prefer:
DB bulk fetch / JOIN / batch query
        ↓
In-memory stream processing
```

---

# 21. Scenario — Parallel Stream Decision ⭐⭐⭐⭐⭐

Never answer:

> “Parallel stream is faster because there are many records.”

Better answer:

```text
Check:
1. Dataset size
2. CPU cost per element
3. Splitting cost
4. Ordering requirements
5. Shared mutable state
6. Blocking I/O
7. Common ForkJoinPool contention
8. Downstream resource limits
9. Benchmark results
```

### Example CPU-bound candidate

```java
long count = employees.parallelStream()
        .filter(this::expensiveCpuCheck)
        .count();
```

Still benchmark before adopting it.

---

# 22. Scenario — Blocking I/O + Parallel Stream ❌

Avoid assuming this is a safe optimization:

```java
employees.parallelStream()
        .map(e -> remoteService.fetch(e.getId()))
        .collect(Collectors.toList());
```

Potential problems:

```text
common pool saturation
remote-service overload
unpredictable latency
connection-pool exhaustion
```

For I/O concurrency, consider a purpose-built executor and explicit limits.

---

# 23. Scenario — Side Effects + Parallel Stream ❌🔥

### Dangerous

```java
List<String> result = new ArrayList<>();

employees.parallelStream()
        .map(Employee::getName)
        .forEach(result::add);
```

`ArrayList` is not thread-safe.

### Better

```java
List<String> result = employees.parallelStream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Prefer collectors designed for the aggregation instead of manually mutating shared state.

---

# 24. Scenario — `orElse()` vs `orElseGet()`

```java
String value = optional.orElse(expensiveFallback());
```

The fallback expression is evaluated eagerly.

```java
String value = optional.orElseGet(this::expensiveFallback);
```

The supplier is invoked only when the value is absent.

---

# 25. Scenario — `toList()` vs `Collectors.toList()`

Modern Java:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .toList();
```

`Stream.toList()` returns an unmodifiable list.

If you need an explicitly mutable `ArrayList`:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toCollection(ArrayList::new));
```

Do not assume `Collectors.toList()` guarantees a particular mutable list implementation.

---

# 26. Scenario — `distinct()` and Equality

```java
List<Employee> unique = employees.stream()
        .distinct()
        .collect(Collectors.toList());
```

`distinct()` depends on `equals()` and `hashCode()`.

Interview question:

> “What happens if Employee doesn't implement logical equality correctly?”

Answer:

> “`distinct()` may not remove the logical duplicates I expect because it uses the object's equality semantics.”

---

# 27. Scenario — Complex Business Rule

### ❌ Giant lambda

```java
employees.stream()
        .filter(e -> e.isActive()
                && e.getExperience() >= 5
                && e.getSalary() > 100000
                && e.getDepartment() != null
                && !e.getDepartment().equals("HR"))
        .collect(Collectors.toList());
```

### ✅ Named domain method

```java
employees.stream()
        .filter(this::isPromotionEligible)
        .collect(Collectors.toList());
```

```java
private boolean isPromotionEligible(Employee e) {
    return e.isActive()
            && e.getExperience() >= 5
            && e.getSalary() > 100000
            && e.getDepartment() != null
            && !e.getDepartment().equals("HR");
}
```

---

# 28. Scenario — Short-Circuiting 🔥

```java
boolean found = employees.stream()
        .filter(Employee::isActive)
        .anyMatch(e -> e.getSalary() > 200000);
```

Once the required result is known, the pipeline may stop processing further elements.

Know these:

```text
anyMatch()
allMatch()
noneMatch()
findFirst()
findAny()
limit()
```

---

# 29. Scenario — Don't Sort When You Don't Need To

### Find minimum

```java
Employee lowest = employees.stream()
        .min(Comparator.comparingInt(Employee::getSalary))
        .orElse(null);
```

### Find maximum

```java
Employee highest = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary))
        .orElse(null);
```

Sorting is useful when you actually need ordered results.

---

# 30. Scenario — Ordering and Parallel Streams

```java
List<String> names = employees.parallelStream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Be careful when interview requirements explicitly demand encounter order.

If ordering does not matter and the operation permits it:

```java
long active = employees.parallelStream()
        .unordered()
        .filter(Employee::isActive)
        .count();
```

`unordered()` is not a magic performance switch. It only removes ordering constraints where the stream semantics allow it.

---

# 31. Complete Runnable Interview Practice Code 💻

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class StreamInterviewFinalRevision {

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
                        Arrays.asList("Java", "Docker")),
                new Employee(6, "Vikas", "Finance", 130000, 8, true,
                        Arrays.asList("SQL", "AWS"))
        );

        // 1. Highest salary
        Employee highest = employees.stream()
                .max(Comparator.comparingInt(Employee::getSalary))
                .orElse(null);

        // 2. Second-highest distinct salary
        Optional<Integer> secondHighest = employees.stream()
                .map(Employee::getSalary)
                .distinct()
                .sorted(Comparator.reverseOrder())
                .skip(1)
                .findFirst();

        // 3. Group by department
        Map<String, List<Employee>> byDepartment = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment));

        // 4. Average salary by department
        Map<String, Double> averageSalaryByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.averagingInt(Employee::getSalary)
                ));

        // 5. Highest paid employee per department
        Map<String, Optional<Employee>> highestByDepartment = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDepartment,
                        Collectors.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 6. Safe toMap with duplicate-key handling
        Map<String, Employee> employeeByDepartment = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getDepartment,
                        Function.identity(),
                        (e1, e2) -> e1
                ));

        // 7. Flatten skills
        List<String> skills = employees.stream()
                .flatMap(e -> e.getSkills().stream())
                .distinct()
                .sorted()
                .collect(Collectors.toList());

        // 8. Partition active/inactive
        Map<Boolean, List<Employee>> activePartition = employees.stream()
                .collect(Collectors.partitioningBy(Employee::isActive));

        // 9. Any employee above threshold
        boolean exists = employees.stream()
                .anyMatch(e -> e.getSalary() > 200000);

        // 10. All employees active
        boolean allActive = employees.stream()
                .allMatch(Employee::isActive);

        // 11. First active employee
        Optional<Employee> firstActive = employees.stream()
                .filter(Employee::isActive)
                .findFirst();

        // 12. Total salary
        int totalSalary = employees.stream()
                .mapToInt(Employee::getSalary)
                .sum();

        // 13. Duplicate salary values
        Map<Integer, Long> salaryFrequency = employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getSalary,
                        Collectors.counting()
                ));

        Set<Integer> duplicateSalaries = salaryFrequency.entrySet().stream()
                .filter(entry -> entry.getValue() > 1)
                .map(Map.Entry::getKey)
                .collect(Collectors.toSet());

        // 14. Promotion eligibility
        List<Employee> promotionEligible = employees.stream()
                .filter(StreamInterviewFinalRevision::isPromotionEligible)
                .collect(Collectors.toList());

        // 15. Parallel example — benchmark before production use
        long activeCountParallel = employees.parallelStream()
                .unordered()
                .filter(Employee::isActive)
                .count();

        System.out.println("Highest: " + highest);
        System.out.println("Second highest salary: " + secondHighest);
        System.out.println("By department: " + byDepartment);
        System.out.println("Average salary: " + averageSalaryByDepartment);
        System.out.println("Highest by department: " + highestByDepartment);
        System.out.println("Employee by department: " + employeeByDepartment);
        System.out.println("Skills: " + skills);
        System.out.println("Active partition: " + activePartition);
        System.out.println("Salary > 200K exists: " + exists);
        System.out.println("All active: " + allActive);
        System.out.println("First active: " + firstActive);
        System.out.println("Total salary: " + totalSalary);
        System.out.println("Duplicate salaries: " + duplicateSalaries);
        System.out.println("Promotion eligible: " + promotionEligible);
        System.out.println("Parallel active count: " + activeCountParallel);

        // Stream laziness demo
        employees.stream()
                .filter(e -> {
                    System.out.println("Filtering: " + e.getName());
                    return e.getSalary() > 100000;
                })
                .map(Employee::getName);

        System.out.println("Lazy pipeline created; no terminal operation yet.");

        employees.stream()
                .filter(e -> e.getSalary() > 100000)
                .map(Employee::getName)
                .forEach(System.out::println);
    }

    private static boolean isPromotionEligible(Employee e) {
        return e.isActive()
                && e.getExperience() >= 5
                && e.getSalary() > 100000
                && e.getDepartment() != null
                && !e.getDepartment().equals("HR");
    }
}
```

---

# 32. Rapid-Fire Interview Questions 🔥

### Fundamentals

1. What is a Stream?
2. Stream vs Collection?
3. Why are streams lazy?
4. Can a stream be reused?
5. Intermediate vs terminal operation?
6. What is a short-circuiting operation?
7. What is a stateful intermediate operation?
8. What does non-interference mean?

### Transformations

9. `map()` vs `flatMap()`?
10. `filter()` vs `map()`?
11. Why use `distinct()`?
12. What does `sorted()` do internally at a high level?
13. When would you use `limit()`?
14. When is `skip()` useful?

### Collectors

15. `groupingBy()` vs `partitioningBy()`?
16. `toMap()` duplicate-key behavior?
17. How does the merge function work?
18. `mapping()` collector?
19. `collectingAndThen()`?
20. `teeing()`?
21. `summarizingInt()`?

### Performance

22. `map()` vs `mapToInt()`?
23. Why avoid unnecessary boxing?
24. Why can `sorted()` be expensive?
25. Why can `distinct()` be expensive?
26. What is short-circuiting?
27. Why can stream pipelines allocate intermediate objects?

### Parallel Streams

28. How does `parallelStream()` work at a high level?
29. What is the common ForkJoinPool?
30. `findFirst()` vs `findAny()` in parallel execution?
31. Why is shared mutable state dangerous?
32. Why is blocking I/O problematic in parallel streams?
33. When should you use `unordered()`?
34. Why should parallel streams be benchmarked?

### Production

35. How can streams cause N+1 database queries?
36. Why shouldn't `peek()` contain business logic?
37. When is a loop better than a stream?
38. How do you test a stream-heavy business rule?
39. How do you debug a complex stream pipeline?
40. What would make you reject a proposed stream optimization?

---

# 33. 15 Must-Know Coding Questions for 5-Year Interviews 💻

1. Find the second-highest distinct salary.
2. Find the highest-paid employee in every department.
3. Find average salary by department.
4. Find duplicate employee IDs.
5. Find duplicate salaries.
6. Group employees by department and count them.
7. Partition active and inactive employees.
8. Find the first employee matching multiple conditions.
9. Flatten all employee skills and return unique sorted skills.
10. Convert employees to a map with duplicate-key handling.
11. Find the department with the highest average salary.
12. Find the top 3 salaries without creating unnecessary intermediate collections.
13. Find employees whose salary is above their department average.
14. Build a frequency map of words using streams.
15. Review a parallel stream and identify thread-safety/performance problems.

---

# 34. Production Code Review Checklist ✅

Before approving a stream pipeline, ask:

```text
□ Is the pipeline readable?
□ Is a loop simpler?
□ Are intermediate operations lazy and correctly ordered?
□ Is there unnecessary sorting?
□ Can a short-circuit operation be used?
□ Is primitive stream usage appropriate?
□ Are there accidental boxing costs?
□ Are side effects explicit and safe?
□ Is the source being modified?
□ Is equals/hashCode correct for distinct()?
□ Can toMap() receive duplicate keys?
□ Are null collections handled?
□ Is there an N+1 database pattern?
□ Is parallelStream() actually justified?
□ Is blocking I/O involved?
□ Is ordering required?
□ Has performance been measured rather than assumed?
```

---

# 35. 2-Minute Final Interview Script 🎤

> “I use the Stream API mainly to express transformations, filtering and aggregation declaratively. I keep readability as the first priority and don't force complex imperative workflows into streams. I remember that intermediate operations are lazy and streams are single-use. For one-to-one transformations I use `map()`, for flattening nested data I use `flatMap()`, and for numeric aggregation I prefer primitive streams such as `mapToInt()`. For existence checks I use short-circuiting operations like `anyMatch()`, and for min/max I use `min()` or `max()` instead of sorting the complete data set. With collectors I pay attention to duplicate keys in `toMap()`, grouping requirements, and equality semantics for `distinct()`. I avoid shared mutable state and business side effects inside `map()` or `peek()`. For parallel streams, I don't assume parallel means faster. I consider dataset size, CPU cost, splitting overhead, ordering, thread safety, blocking I/O, common-pool contention and downstream resource limits, and I benchmark before making a production decision. Finally, I always consider whether a simple loop would be clearer.”

---

# 36. Final Stream API Revision 🏁

```text
Source
  ↓
stream()
  ↓
filter()
  ↓
map() / flatMap()
  ↓
sorted() / distinct() / limit() / skip()
  ↓
collect() / reduce() / count() / find() / match()
```

### Golden rules

```text
1. Streams are not collections.
2. Intermediate operations are lazy.
3. Streams are single-use.
4. map() transforms.
5. flatMap() flattens.
6. filter() selects.
7. collect() aggregates.
8. reduce() performs reduction.
9. Prefer short-circuiting when possible.
10. Prefer primitive streams for numeric work.
11. Avoid side effects.
12. Don't mutate the source.
13. Don't blindly use parallelStream().
14. Watch for N+1 database calls.
15. Optimize only after measuring.
16. Readability beats cleverness.
```

---

# 🧪 Complete Practice Code

[GitHub — 9.23 Stream API Interview Scenarios & Final Revision Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/23-Stream-API-Interview-Scenarios-and-Final-Revision)

---

## Navigation

[← 9.22 — Stream API Best Practices & Anti-Patterns](../22-Stream-API-Best-Practices-and-Anti-Patterns/README.md)

**Current → 9.23 — Stream API Interview Scenarios & Final Revision → ✅ Completed**

**Chapter 9 Stream API → Final Revision Complete ✅**