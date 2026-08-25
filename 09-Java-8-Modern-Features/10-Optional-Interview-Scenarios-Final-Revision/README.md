# 9.10 — Optional Interview Scenarios & Final Revision

> **Interview Goal:** Finish Optional at 5-year Java interview level by combining `ofNullable()`, `map()`, `flatMap()`, `filter()`, `orElse()`, `orElseGet()`, `orElseThrow()`, `ifPresent()`, best practices, anti-patterns, and real production scenarios.

## 🎯 60-Second Master Answer

> `Optional` is a container used to explicitly model the possible absence of a value. I mainly use it as a return type when absence is a valid outcome. `map()` transforms a present value, `flatMap()` is used when the transformation already returns an Optional, and `filter()` keeps the value only when a predicate passes. For fallback handling, `orElse()` evaluates its argument eagerly, `orElseGet()` evaluates a Supplier lazily when the Optional is empty, and `orElseThrow()` is appropriate when absence is exceptional. I avoid blindly calling `get()`, returning null from Optional-returning methods, and using Optional everywhere as a replacement for null.

---

# 1. Optional Interview Decision Tree ⭐⭐⭐⭐⭐

```text
Do I have a possibly-null value?
          │
          ├── yes → ofNullable()
          │
          └── no  → Optional.of()

Need a condition?
          ↓
       filter()

Need normal transformation?
          ↓
        map()

Transformation returns Optional?
          ↓
       flatMap()

Need a result when absent?
          │
          ├── simple value → orElse()
          ├── computed/expensive → orElseGet()
          └── exceptional → orElseThrow()

Need only a side effect?
          ↓
      ifPresent()
```

---

# 2. Scenario — Repository `findById()` ⭐⭐⭐⭐⭐

### Requirement

A repository may or may not find an employee.

```java
interface EmployeeRepository {
    Optional<Employee> findById(long id);
}
```

Good contract:

```java
Optional<Employee> findById(long id)
```

Missing employee:

```java
return Optional.empty();
```

Never:

```java
return null;
```

---

# 3. Scenario — Service Throws When Missing ⭐⭐⭐⭐⭐

```java
public Employee getEmployee(long id) {
    return repository.findById(id)
            .orElseThrow(() ->
                    new EmployeeNotFoundException(
                            "Employee not found: " + id));
}
```

### Interview explanation

> The repository models absence using Optional. The service decides that absence is exceptional for this operation and converts it into a domain exception.

---

# 4. Scenario — Return Default When Missing

```java
public String getEmployeeName(long id) {
    return repository.findById(id)
            .map(Employee::getName)
            .orElse("Unknown Employee");
}
```

Use this when missing data is a normal business case.

---

# 5. Scenario — Nested Employee → Address → City ⭐⭐⭐⭐⭐

Assume:

```java
Employee::getAddress → Optional<Address>
Address::getCity     → String
```

Correct:

```java
public String getEmployeeCity(long id) {
    return repository.findById(id)
            .flatMap(Employee::getAddress)
            .map(Address::getCity)
            .orElse("Unknown City");
}
```

### Why `flatMap()`?

Because:

```java
Employee::getAddress
```

returns:

```java
Optional<Address>
```

If `map()` were used:

```java
Optional<Optional<Address>>
```

would result.

---

# 6. Scenario — Business Eligibility ⭐⭐⭐⭐⭐

Requirement:

> Return employee name only when salary is above 10 LPA.

```java
String result = employeeOptional
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .orElse("Not Eligible");
```

Flow:

```text
Employee
   ↓
filter salary
   ↓
Employee
   ↓
map name
   ↓
String
   ↓
orElse fallback
```

---

# 7. Scenario — Cache + Database ⭐⭐⭐⭐⭐

Requirement:

> Use the cached user if available; otherwise load from database.

```java
User user = cachedUser
        .orElseGet(() -> database.load(id));
```

Why not:

```java
User user = cachedUser
        .orElse(database.load(id));
```

Because the database expression is evaluated eagerly with `orElse()`.

### Interview answer

> For expensive or dynamic fallback computation, I prefer `orElseGet()` because the Supplier is invoked only when the Optional is empty.

---

# 8. Scenario — Present Optional and Eager Evaluation 🔥

```java
static String createDefault() {
    System.out.println("DEFAULT CREATED");
    return "DEFAULT";
}

Optional<String> value = Optional.of("JAVA");

String result = value.orElse(createDefault());
```

Output:

```text
DEFAULT CREATED
```

But:

```java
result.equals("JAVA")
```

is `true`.

Important distinction:

```text
fallback expression evaluated → YES
fallback returned              → NO
```

---

# 9. Scenario — Same Case with `orElseGet()`

```java
Optional<String> value = Optional.of("JAVA");

String result = value.orElseGet(() -> createDefault());
```

`createDefault()` is not invoked.

```text
Optional present
      ↓
existing value returned
      ↓
Supplier not invoked
```

---

# 10. Scenario — `ifPresent()` for Side Effects

Requirement:

> Send an audit event only when an employee exists.

```java
employeeOptional.ifPresent(
        employee -> auditService.log(employee.getId())
);
```

Do not unnecessarily write:

```java
if (employeeOptional.isPresent()) {
    auditService.log(employeeOptional.get().getId());
}
```

The first version expresses the intent directly.

---

# 11. Scenario — Java 9 `ifPresentOrElse()`

```java
employeeOptional.ifPresentOrElse(
        employee -> System.out.println(employee.getName()),
        () -> System.out.println("Employee not found")
);
```

Remember:

```text
ifPresent()     → present action
ifPresentOrElse → present + absent actions
```

`ifPresentOrElse()` was introduced in Java 9, not Java 8.

---

# 12. Scenario — Nullable External API

Suppose a legacy API returns null:

```java
String name = legacyApi.getName();
```

Create the Optional boundary:

```java
Optional<String> optionalName =
        Optional.ofNullable(name);
```

Then:

```java
String normalized = optionalName
        .filter(value -> !value.isBlank())
        .map(String::trim)
        .orElse("Unknown");
```

For Java 8, replace `isBlank()` with an appropriate Java 8-compatible check because `String.isBlank()` was added in Java 11.

---

# 13. Scenario — Avoid `Optional.of(null)`

Wrong:

```java
Optional<String> value = Optional.of(nullableName);
```

if `nullableName` can be null.

Correct:

```java
Optional<String> value = Optional.ofNullable(nullableName);
```

### Memory

```text
of(null)          → NullPointerException
ofNullable(null)  → Optional.empty()
```

---

# 14. Scenario — Optional Method Must Not Return null

Bad:

```java
Optional<Employee> findEmployee(long id) {
    if (id <= 0) {
        return null;
    }
    return ...;
}
```

Correct:

```java
Optional<Employee> findEmployee(long id) {
    if (id <= 0) {
        return Optional.empty();
    }
    return ...;
}
```

### Interview line

> An Optional-returning method should return an Optional object in all paths; absence is represented by `Optional.empty()`, not null.

---

# 15. Scenario — Remove `isPresent()` + `get()`

### Before

```java
if (employeeOptional.isPresent()) {
    Employee employee = employeeOptional.get();
    System.out.println(employee.getName());
}
```

### After

```java
employeeOptional
        .map(Employee::getName)
        .ifPresent(System.out::println);
```

This is shorter and makes the transformation explicit.

---

# 16. Scenario — `map()` vs `flatMap()` Interview Trap 🔥

Given:

```java
Optional<Employee> employee;
```

and:

```java
Optional<Address> getAddress(Employee employee)
```

Wrong:

```java
Optional<Optional<Address>> address =
        employee.map(this::getAddress);
```

Correct:

```java
Optional<Address> address =
        employee.flatMap(this::getAddress);
```

### One-line rule

```text
Function returns R           → map()
Function returns Optional<R> → flatMap()
```

---

# 17. Scenario — Multiple Conditions

```java
String result = employeeOptional
        .filter(Employee::isActive)
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .orElse("Not Eligible");
```

Both conditions must pass.

If either fails:

```text
Optional.empty()
```

---

# 18. Scenario — Optional + Stream ⭐⭐⭐⭐⭐

Java 9+:

```java
List<String> managerNames = employees.stream()
        .map(Employee::getManager)
        .flatMap(Optional::stream)
        .map(Employee::getName)
        .toList();
```

Java 8:

```java
List<String> managerNames = employees.stream()
        .map(Employee::getManager)
        .filter(Optional::isPresent)
        .map(Optional::get)
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Interview point:

> `Optional.stream()` was introduced in Java 9 and is useful for flattening present Optional values into a Stream.

---

# 19. Scenario — `Optional<Collection<T>>`

Usually avoid:

```java
Optional<List<Employee>> findEmployees(...)
```

when:

```text
empty list = no employees
```

Prefer:

```java
List<Employee> findEmployees(...)
```

and return:

```java
Collections.emptyList()
```

### Nuance

If the domain distinguishes:

```text
collection absent
vs
collection exists but is empty
```

then Optional may be justified.

---

# 20. Scenario — Optional Field / Parameter

Avoid by default:

```java
class Employee {
    private Optional<String> phone;
}
```

and:

```java
void update(Optional<String> phone) {}
```

Instead, model the domain semantics explicitly and use Optional at suitable API boundaries.

### Interview nuance

> These are design guidelines, not compiler restrictions. The key consideration is whether Optional makes the contract clearer or merely adds wrapper complexity.

---

# 21. Scenario — `orElseThrow()`

Modern Java:

```java
Employee employee = employeeOptional.orElseThrow();
```

This no-argument overload was added in Java 10.

Java 8 style:

```java
Employee employee = employeeOptional.orElseThrow(
        () -> new EmployeeNotFoundException("Employee not found"));
```

Interview trap:

```text
Java 8 → supplier-based orElseThrow()
Java 10+ → no-argument orElseThrow()
```

---

# 22. Scenario — Primitive Optional Types

For primitive values, Java provides:

```java
OptionalInt
OptionalLong
OptionalDouble
```

Example:

```java
OptionalInt salary = OptionalInt.of(150000);

int value = salary.orElse(0);
```

These avoid wrapping primitive values in `Optional<Integer>`, `Optional<Long>`, etc. when the primitive-specialized API is appropriate.

---

# 23. Scenario — Avoid `orElse(null)` as a Habit

Technically valid:

```java
String value = optional.orElse(null);
```

But if the surrounding API can work with Optional, consider keeping the absence explicit instead of immediately converting it back to null.

Use it only when a legacy/external contract specifically requires null.

---

# 24. Scenario — Side Effects in `map()`

Avoid hiding side effects:

```java
optional.map(employee -> {
    audit(employee);
    return employee.getName();
});
```

Prefer:

```java
optional.ifPresent(this::audit);

Optional<String> name =
        optional.map(Employee::getName);
```

### Principle

```text
map()       → transformation
ifPresent() → side effect
```

---

# 25. Complete Interview-Level Program ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;

class Employee {

    private final long id;
    private final String name;
    private final double salary;
    private final boolean active;
    private final Address address;

    Employee(long id, String name, double salary,
             boolean active, Address address) {
        this.id = id;
        this.name = name;
        this.salary = salary;
        this.active = active;
        this.address = address;
    }

    public long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }

    public boolean isActive() {
        return active;
    }

    public Optional<Address> getAddress() {
        return Optional.ofNullable(address);
    }
}

class Address {

    private final String city;

    Address(String city) {
        this.city = city;
    }

    public String getCity() {
        return city;
    }
}

public class OptionalInterviewFinalDemo {

    static Optional<Employee> findEmployee(long id) {
        if (id == 101) {
            return Optional.of(
                    new Employee(
                            101,
                            "Nirbhay",
                            150000,
                            true,
                            new Address("Bangalore")));
        }

        if (id == 102) {
            return Optional.of(
                    new Employee(
                            102,
                            "Rahul",
                            80000,
                            true,
                            null));
        }

        return Optional.empty();
    }

    static String createDefault() {
        System.out.println("Creating expensive default...");
        return "Unknown";
    }

    public static void main(String[] args) {

        Optional<Employee> employee = findEmployee(101);

        // 1. map()
        String name = employee
                .map(Employee::getName)
                .orElse("Unknown Employee");

        // 2. filter() + map()
        String eligibleName = employee
                .filter(Employee::isActive)
                .filter(e -> e.getSalary() > 100000)
                .map(Employee::getName)
                .orElse("Not Eligible");

        // 3. flatMap() + map()
        String city = employee
                .flatMap(Employee::getAddress)
                .map(Address::getCity)
                .orElse("Unknown City");

        // 4. orElse() is eager
        String eager = employee
                .map(Employee::getName)
                .orElse(createDefault());

        // 5. orElseGet() is lazy
        String lazy = employee
                .map(Employee::getName)
                .orElseGet(OptionalInterviewFinalDemo::createDefault);

        // 6. ifPresent()
        employee.ifPresent(e ->
                System.out.println("Audit employee: " + e.getId()));

        // 7. Missing employee
        Optional<Employee> missing = findEmployee(999);

        String missingName = missing
                .map(Employee::getName)
                .orElse("Employee Not Found");

        // 8. orElseThrow()
        Employee required = findEmployee(101)
                .orElseThrow(() ->
                        new IllegalStateException("Employee not found"));

        System.out.println("Name: " + name);
        System.out.println("Eligible: " + eligibleName);
        System.out.println("City: " + city);
        System.out.println("Eager: " + eager);
        System.out.println("Lazy: " + lazy);
        System.out.println("Missing: " + missingName);
        System.out.println("Required: " + required.getName());
    }
}
```

---

# 26. Predict the Output — Interview Test 🔥

```java
Optional<String> value = Optional.of("Java");

String result = value.orElse(printDefault());
```

Assume:

```java
static String printDefault() {
    System.out.println("DEFAULT");
    return "Default";
}
```

### Answer

Output:

```text
DEFAULT
```

But:

```text
result = Java
```

Reason:

```text
method argument evaluated first
        ↓
printDefault() executes
        ↓
orElse receives "Default"
        ↓
Optional already contains "Java"
        ↓
"Java" returned
```

---

# 27. Predict the Output — `orElseGet()` 🔥

```java
Optional<String> value = Optional.of("Java");

String result = value.orElseGet(() -> printDefault());
```

Output:

```text
nothing from printDefault()
```

Result:

```text
Java
```

Reason:

```text
Optional present
      ↓
return existing value
      ↓
Supplier not invoked
```

---

# 28. Predict the Output — `filter()` 🔥

```java
Optional<Integer> value = Optional.of(50);

Optional<Integer> result = value
        .filter(x -> x > 100);

System.out.println(result);
```

Output:

```text
Optional.empty
```

Reason:

```text
50 > 100 → false
          ↓
Optional.empty()
```

---

# 29. Predict the Type — `map()` vs `flatMap()` 🔥

Given:

```java
Optional<Employee> employee;
```

and:

```java
Optional<Address> getAddress(Employee e)
```

Then:

```java
employee.map(this::getAddress)
```

has type:

```java
Optional<Optional<Address>>
```

while:

```java
employee.flatMap(this::getAddress)
```

has type:

```java
Optional<Address>
```

This is one of the highest-value Optional interview questions.

---

# 30. Top 20 Optional Interview Questions 🔥🔥🔥

### Q1. What is Optional?

A container representing a value that may be present or absent.

### Q2. Why was Optional introduced?

Primarily to make absence explicit in APIs and reduce accidental null handling problems.

### Q3. Is Optional a replacement for null everywhere?

No.

### Q4. Where should Optional generally be used?

Most commonly as a return type where absence is meaningful.

### Q5. `of()` vs `ofNullable()`?

`of()` rejects null; `ofNullable()` converts null to empty.

### Q6. What does `filter()` do?

Keeps the value only if the Predicate passes.

### Q7. `map()` vs `flatMap()`?

`map()` handles normal values; `flatMap()` handles Optional-returning transformations.

### Q8. `orElse()` vs `orElseGet()`?

Eager fallback evaluation vs lazy Supplier execution.

### Q9. When use `orElseThrow()`?

When absence should result in an exception.

### Q10. Why avoid `get()`?

It can throw `NoSuchElementException` when empty.

### Q11. Is `isPresent()` bad?

No, but `isPresent()` + `get()` is often less expressive than direct Optional operations.

### Q12. Can an Optional contain null?

No.

### Q13. Can an Optional-returning method return null?

It technically can, but it violates the API contract and should not.

### Q14. What is `Optional.empty()`?

An Optional with no value.

### Q15. Why avoid Optional as a field?

It often adds wrapper complexity and can complicate framework/serialization usage.

### Q16. Why avoid Optional parameters?

They can make API contracts less clear; domain-specific parameters or request types are often better.

### Q17. Why usually avoid Optional collection wrappers?

An empty collection already communicates “no elements” in many APIs.

### Q18. What is `Optional.stream()`?

Java 9 API that turns a present Optional into a one-element stream and an empty Optional into an empty stream.

### Q19. What are `OptionalInt`, `OptionalLong`, and `OptionalDouble`?

Primitive-specialized Optional types.

### Q20. What is the main design principle?

Use Optional where it makes absence explicit and the API clearer, not everywhere.

---

# 31. 10 Coding Challenges 🔥

### Challenge 1
Implement:

```java
Optional<Employee> findById(long id)
```

without ever returning null.

### Challenge 2
Convert:

```java
if (employee.isPresent()) {
    System.out.println(employee.get().getName());
}
```

to `ifPresent()`.

### Challenge 3
Get city from:

```text
Employee → Optional<Address> → city
```

using `flatMap()` + `map()`.

### Challenge 4
Filter employees with:

```text
active == true
salary > 10 LPA
```

and return the name.

### Challenge 5
Demonstrate `orElse()` eager evaluation.

### Challenge 6
Demonstrate `orElseGet()` lazy evaluation.

### Challenge 7
Build a cache → database fallback using `orElseGet()`.

### Challenge 8
Convert `List<Optional<Employee>>` into `List<Employee>`.

### Challenge 9
Find the first matching employee using a Stream and return `Optional<Employee>`.

### Challenge 10 — Final ⭐⭐⭐⭐⭐
Build:

```text
Repository
   ↓
Optional<Employee>
   ↓
filter active
   ↓
filter salary
   ↓
flatMap address
   ↓
map city
   ↓
orElseGet fallback
```

Then explain every operation in under 60 seconds.

---

# 32. Optional Cheat Sheet 🧠

```text
CREATE
────────────────────────
Optional.of(value)
Optional.ofNullable(value)
Optional.empty()
```

```text
TRANSFORM
────────────────────────
map()
flatMap()
```

```text
FILTER
────────────────────────
filter()
```

```text
CONSUME
────────────────────────
ifPresent()
ifPresentOrElse()   // Java 9+
```

```text
FALLBACK
────────────────────────
orElse()             // eager
orElseGet()         // lazy
orElseThrow()       // exception
```

```text
PRIMITIVES
────────────────────────
OptionalInt
OptionalLong
OptionalDouble
```

---

# 33. The Ultimate Optional Flow ⭐⭐⭐⭐⭐

```text
Nullable source
      ↓
  ofNullable()
      ↓
  Optional<T>
      ↓
    filter()
      ↓
  Optional<T>
      ↓
    map()
      ↓
  Optional<R>
      ↓
  flatMap()   ← when next function returns Optional<R>
      ↓
  Optional<X>
      ↓
 ┌────┴──────────────┐
 ↓                   ↓
normal             exceptional
 ↓                   ↓
orElse /             orElseThrow
orElseGet
```

---

# 34. Final Interview Rules — Memorize These 🔥

```text
1. Optional is mainly useful for return types.
2. Optional.of(null) throws NullPointerException.
3. Optional.ofNullable(null) = Optional.empty().
4. Never return null from an Optional-returning method.
5. map() → normal transformation.
6. flatMap() → Optional-returning transformation.
7. filter() → keep/discard based on Predicate.
8. orElse() → eager fallback expression.
9. orElseGet() → lazy Supplier fallback.
10. orElseThrow() → absence is exceptional.
11. get() can throw NoSuchElementException.
12. isPresent() + get() is often replaceable by direct Optional operations.
13. Optional is not a universal replacement for null.
14. Avoid Optional fields/parameters by default.
15. Prefer empty collections when absence simply means zero elements.
16. Keep Optional pipelines readable.
17. Separate transformations from side effects.
18. Know Java-version differences such as Optional.stream() and ifPresentOrElse().
19. Use primitive Optional types when appropriate.
20. Choose Optional based on API clarity and domain semantics.
```

---

# 35. Final 2-Minute Interview Script 🎤

> “Optional was introduced in Java 8 to represent a value that may be absent and make that absence explicit in APIs. I generally use it as a return type, for example a repository `findById()` method returning `Optional<Employee>`. I use `ofNullable()` when converting a potentially null value into Optional and `of()` only when the value is guaranteed non-null. For processing, `filter()` applies a condition, `map()` transforms a value, and `flatMap()` is used when the transformation already returns Optional. For absence handling, `orElse()` gives a fallback but evaluates the fallback expression eagerly, whereas `orElseGet()` uses a Supplier and evaluates it lazily. `orElseThrow()` is appropriate when missing data is exceptional. I avoid blindly calling `get()`, returning null from Optional methods, and using Optional as a field or parameter unless the API design specifically benefits from it. The goal is not to eliminate every null; it is to make meaningful absence explicit while keeping the code readable.”

---

# 🧪 Complete Practice Code

urlGitHub — 9.10 Optional Interview Scenarios & Final Revision Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/10-Optional-Interview-Scenarios-Final-Revision

---

## Navigation

[← 9.9 — Optional Best Practices & Anti-Patterns](../09-Optional-Best-Practices-Anti-Patterns/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.10 — Optional Interview Scenarios & Final Revision → ✅ Completed**

**Chapter 9 Optional Section → ✅ Completed**

**Next → 9.11 — Java 8 Stream API Fundamentals**