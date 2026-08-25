# 9.9 — Optional Best Practices & Anti-Patterns

> **Interview Goal:** Know where `Optional` improves Java code, where it should NOT be used, and how to write production-quality Optional pipelines without turning simple logic into unreadable code.

## 🎯 30-Second Interview Answer

> `Optional` is primarily a return-type abstraction for representing the possible absence of a value. Good practice is to use it at API boundaries where absence is meaningful, compose it with `map`, `flatMap`, `filter`, and appropriate fallback methods, and avoid using it as a field, method parameter, collection element, or replacement for every null check. `Optional` should make absence explicit without making the code harder to understand.

---

# 1. What Problem Does Optional Solve?

Before Java 8:

```java
Employee employee = findEmployee(id);

if (employee != null) {
    System.out.println(employee.getName());
}
```

With Optional:

```java
Optional<Employee> employee = findEmployee(id);

employee.map(Employee::getName)
        .ifPresent(System.out::println);
```

The API communicates:

```text
Employee may be absent
```

instead of hiding that possibility in a nullable return value.

---

# 2. Best Practice #1 — Use Optional Mainly as a Return Type ⭐⭐⭐⭐⭐

Good:

```java
public Optional<Employee> findById(long id) {
    // lookup
}
```

The caller is forced to consider absence.

Example:

```java
Optional<Employee> employee = employeeService.findById(101);

String name = employee
        .map(Employee::getName)
        .orElse("Unknown");
```

### Interview line

> Optional is most commonly useful as a method return type when the absence of a result is a normal and meaningful outcome.

---

# 3. Anti-Pattern #1 — Optional as a Field ❌

Avoid:

```java
class Employee {
    private Optional<String> phoneNumber;
}
```

Why?

- Adds unnecessary object/state complexity.
- Makes serialization and framework integration more awkward in many environments.
- A field can generally be represented with a nullable value and handled at the API boundary.
- Optional was not designed as a general-purpose replacement for every nullable field.

Prefer:

```java
class Employee {
    private String phoneNumber;
}
```

Then expose absence deliberately when appropriate:

```java
public Optional<String> getPhoneNumber() {
    return Optional.ofNullable(phoneNumber);
}
```

---

# 4. Anti-Pattern #2 — Optional as a Method Parameter ❌

Avoid:

```java
void sendEmail(Optional<String> email) {
}
```

This creates ambiguity:

```text
sendEmail(Optional.of("a@b.com"))
        ↓
clear enough

sendEmail(Optional.empty())
        ↓
what does empty mean?
```

Often better:

```java
void sendEmail(String email) {
}
```

or, if the operation has two clearly different behaviors, expose separate methods or a domain-specific request type.

### Interview nuance

The key issue is API clarity, not that passing Optional is technically impossible.

---

# 5. Anti-Pattern #3 — Calling `get()` Without Checking ❌

Bad:

```java
Optional<Employee> employee = findEmployee(101);

Employee result = employee.get();
```

If empty:

```text
NoSuchElementException
```

Prefer:

```java
Employee result = employee
        .orElseThrow(() ->
                new EmployeeNotFoundException("Employee not found"));
```

Or, if absence is normal:

```java
String name = employee
        .map(Employee::getName)
        .orElse("Unknown");
```

### Important

`Optional.get()` is not forbidden in every situation, but it is usually a code smell when absence is genuinely possible and has not been established beforehand.

---

# 6. Anti-Pattern #4 — `isPresent()` + `get()` ❌

This is common but often defeats the purpose of Optional:

```java
if (employee.isPresent()) {
    System.out.println(employee.get().getName());
}
```

Prefer:

```java
employee.map(Employee::getName)
        .ifPresent(System.out::println);
```

Or:

```java
String name = employee
        .map(Employee::getName)
        .orElse("Unknown");
```

### Interview point

`isPresent()` itself is not bad. The problem is using `isPresent()` immediately followed by `get()` when the pipeline operations already express the intended behavior.

---

# 7. Anti-Pattern #5 — `Optional.of(null)` ❌

Never:

```java
Optional<String> value = Optional.of(null);
```

It throws:

```text
NullPointerException
```

Use:

```java
Optional<String> value = Optional.ofNullable(nullableValue);
```

Remember:

```text
of()         → null not allowed
ofNullable() → null becomes empty
```

---

# 8. Anti-Pattern #6 — Returning `null` Instead of Optional ❌

If a method promises:

```java
Optional<Employee> findEmployee(long id)
```

do not return:

```java
return null;
```

Return:

```java
return Optional.empty();
```

### Contract

```text
Optional return type
        ↓
never return null Optional
        ↓
use Optional.empty()
```

---

# 9. Anti-Pattern #7 — `Optional<Collection<T>>` in Most APIs ⚠️

Usually, an empty collection already represents no elements:

```java
List<Employee> findEmployeesByDepartment(String department)
```

Return:

```java
Collections.emptyList()
```

rather than:

```java
Optional<List<Employee>>
```

when the only meaning of absence is “there are no employees.”

### Why?

You avoid two levels of absence:

```text
Optional.empty()

vs

Optional.of(emptyList())
```

Those can introduce unnecessary semantic complexity.

### Nuance

If your domain distinguishes “no collection result exists” from “a collection exists and is empty,” `Optional<Collection<T>>` may be meaningful. The decision should be semantic, not mechanical.

---

# 10. Anti-Pattern #8 — Optional Inside Collections ⚠️

Avoid unnecessarily creating:

```java
List<Optional<Employee>> employees;
```

when the collection can simply contain only valid Employees:

```java
List<Employee> employees;
```

If processing a stream produces Optionals, flatten them deliberately:

```java
List<Employee> employees =
        optionalEmployees.stream()
                .flatMap(Optional::stream)
                .toList();
```

This is especially useful when working with Java 9+.

For Java 8 compatibility, use:

```java
List<Employee> employees =
        optionalEmployees.stream()
                .filter(Optional::isPresent)
                .map(Optional::get)
                .collect(Collectors.toList());
```

---

# 11. Anti-Pattern #9 — Using Optional for Every Null Check ❌

Do not turn simple code into an unnecessarily complicated pipeline.

Over-engineered:

```java
String name = Optional.ofNullable(employee)
        .map(Employee::getName)
        .orElse(null);
```

If the surrounding method already has a clear null contract, a simple check may be easier to read:

```java
String name = employee == null ? null : employee.getName();
```

### Key principle

> Optional is a design tool, not a requirement to eliminate every `null` from every line of Java code.

---

# 12. Anti-Pattern #10 — Optional for Performance-Critical Hot Paths Without Reason ⚠️

Optional is usually perfectly reasonable for API clarity, but do not blindly introduce it into extremely performance-sensitive low-level code without measuring.

For most business applications:

```text
clarity + correctness
        ↓
more important than
        ↓
micro-optimizing Optional allocation
```

For hot paths:

```text
measure → profile → optimize
```

Do not claim that Optional is universally “slow.” The correct engineering answer depends on the workload and JVM behavior.

---

# 13. Best Practice #2 — Use `ofNullable()` at Nullable Boundaries

Example:

```java
String databaseName = resultSet.getString("name");

Optional<String> name =
        Optional.ofNullable(databaseName);
```

This converts a nullable external value into an explicit Optional boundary.

---

# 14. Best Practice #3 — Keep Optional Pipelines Readable ⭐⭐⭐⭐⭐

Good:

```java
String city = employee
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .orElse("Unknown City");
```

Badly over-complicated:

```java
String city = Optional.ofNullable(employee)
        .filter(e -> e != null)
        .map(e -> Optional.ofNullable(e.getAddress()))
        .orElse(Optional.empty())
        .map(a -> a.orElse(null))
        .map(a -> a.getCity())
        .orElse("Unknown");
```

### Rule

```text
Prefer the simplest pipeline that clearly expresses the domain rule.
```

---

# 15. Best Practice #4 — Use `map()` / `flatMap()` Correctly

```java
Optional<Employee> employee = findEmployee(101);
```

Normal value:

```java
Optional<String> name =
        employee.map(Employee::getName);
```

Optional value:

```java
Optional<Address> address =
        employee.flatMap(Employee::getAddress);
```

Golden rule:

```text
R             → map()
Optional<R>   → flatMap()
```

---

# 16. Best Practice #5 — Choose the Correct Fallback

Simple constant:

```java
String name =
        optionalName.orElse("Unknown");
```

Expensive/dynamic fallback:

```java
String name =
        optionalName.orElseGet(() -> loadDefaultName());
```

Exceptional absence:

```java
Employee employee =
        optionalEmployee.orElseThrow(
                () -> new EmployeeNotFoundException("Not found"));
```

Decision tree:

```text
Value exists?
     │
     ├── yes → use value
     │
     └── no
          │
          ├── normal fallback → orElse()
          ├── computed fallback → orElseGet()
          └── exceptional → orElseThrow()
```

---

# 17. Best Practice #6 — Avoid Side Effects in Optional Transformations

Avoid:

```java
optional.map(value -> {
    database.save(value);
    return value.getName();
});
```

`map()` should ideally represent a transformation.

Prefer separating side effects:

```java
optional.ifPresent(database::save);
```

and transformation:

```java
Optional<String> name =
        optional.map(Employee::getName);
```

### Interview nuance

Optional does not prohibit side effects. The recommendation is about readability, predictable pipelines, and separation of concerns.

---

# 18. Best Practice #7 — Use `ifPresentOrElse()` When Both Cases Need Actions

Java 9+:

```java
employee.ifPresentOrElse(
        value -> System.out.println(value.getName()),
        () -> System.out.println("Employee not found")
);
```

Instead of manually writing:

```java
if (employee.isPresent()) {
    System.out.println(employee.get().getName());
} else {
    System.out.println("Employee not found");
}
```

For Java 8, use normal control flow or another appropriate design because `ifPresentOrElse()` was added in Java 9.

---

# 19. Best Practice #8 — Use `Optional.stream()` for Stream Pipelines

Java 9+:

```java
List<String> names =
        employees.stream()
                .map(Employee::getManager)
                .flatMap(Optional::stream)
                .map(Employee::getName)
                .toList();
```

This removes empty Optionals naturally.

### Java 8 version

```java
List<String> names =
        employees.stream()
                .map(Employee::getManager)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .map(Employee::getName)
                .collect(Collectors.toList());
```

---

# 20. Best Practice #9 — Do Not Serialize Optional as a Domain Field by Default

Optional is primarily an API abstraction, not a general-purpose serialization model.

For DTO/entity design, prefer the representation required by your serialization/framework contract and expose Optional at appropriate service/API boundaries when useful.

Example domain field:

```java
private String middleName;
```

API getter, when appropriate:

```java
public Optional<String> getMiddleName() {
    return Optional.ofNullable(middleName);
}
```

Always consider the conventions of the framework you are using.

---

# 21. Best Practice #10 — Do Not Nest Optional Unnecessarily

Avoid:

```java
Optional<Optional<String>> value;
```

Usually this means the API design or chaining operation should be reconsidered.

For Optional-returning functions:

```java
optional.flatMap(this::findValue);
```

instead of:

```java
optional.map(this::findValue);
```

when `findValue()` returns `Optional<String>`.

---

# 22. Real-World Service Example ⭐⭐⭐⭐⭐

Suppose:

```java
interface EmployeeRepository {
    Optional<Employee> findById(long id);
}
```

Service:

```java
public Employee getEmployee(long id) {
    return repository.findById(id)
            .orElseThrow(() ->
                    new EmployeeNotFoundException(
                            "Employee not found: " + id));
}
```

This is clean because the repository communicates possible absence and the service converts that absence into a domain exception at the appropriate boundary.

---

# 23. Real-World Nested Lookup ⭐⭐⭐⭐⭐

```java
public String getEmployeeCity(long id) {
    return repository.findById(id)
            .flatMap(Employee::getAddress)
            .map(Address::getCity)
            .orElse("Unknown City");
}
```

This is preferable to:

```java
Employee employee = repository.findById(id).get();
Address address = employee.getAddress().get();
return address.getCity();
```

because the second version assumes every value exists and can throw `NoSuchElementException`.

---

# 24. Complete Practice Program ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.stream.Collectors;

class Address {

    private final String city;

    Address(String city) {
        this.city = city;
    }

    public String getCity() {
        return city;
    }
}

class Employee {

    private final long id;
    private final String name;
    private final Address address;

    Employee(long id, String name, Address address) {
        this.id = id;
        this.name = name;
        this.address = address;
    }

    public String getName() {
        return name;
    }

    public Optional<Address> getAddress() {
        return Optional.ofNullable(address);
    }

    @Override
    public String toString() {
        return id + " - " + name;
    }
}

public class OptionalBestPracticesDemo {

    static Optional<Employee> findEmployee(long id) {
        if (id == 101) {
            return Optional.of(
                    new Employee(
                            101,
                            "Nirbhay",
                            new Address("Bangalore")));
        }
        return Optional.empty();
    }

    static String loadExpensiveDefault() {
        System.out.println("Expensive fallback executed");
        return "Unknown";
    }

    public static void main(String[] args) {

        // 1. Good return-type usage
        Optional<Employee> employee =
                findEmployee(101);

        // 2. map() for normal transformation
        String name = employee
                .map(Employee::getName)
                .orElse("Unknown Employee");

        // 3. flatMap() for Optional-returning transformation
        String city = employee
                .flatMap(Employee::getAddress)
                .map(Address::getCity)
                .orElse("Unknown City");

        // 4. filter() for a condition
        String eligible = employee
                .filter(e -> e.getName().length() > 5)
                .map(Employee::getName)
                .orElse("Not Eligible");

        // 5. orElse() is eager
        String eager = employee
                .map(Employee::getName)
                .orElse(loadExpensiveDefault());

        // 6. orElseGet() is lazy
        String lazy = employee
                .map(Employee::getName)
                .orElseGet(OptionalBestPracticesDemo::loadExpensiveDefault);

        // 7. Never return null Optional
        Optional<Employee> missing = findEmployee(999);

        System.out.println("Name: " + name);
        System.out.println("City: " + city);
        System.out.println("Eligible: " + eligible);
        System.out.println("Eager: " + eager);
        System.out.println("Lazy: " + lazy);
        System.out.println("Missing: " + missing.orElse(null));
    }
}
```

### Key observation

The eager fallback method executes even though the employee name exists. The `orElseGet()` fallback does not execute in that case.

---

# 25. Anti-Pattern Conversion Exercise 🔥

### Bad code

```java
if (employee.isPresent()) {
    Employee e = employee.get();

    if (e.getAddress().isPresent()) {
        Address a = e.getAddress().get();
        return a.getCity();
    }
}

return "Unknown";
```

### Better

```java
return employee
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .orElse("Unknown");
```

### Interview explanation

> The second version composes optional-returning operations with `flatMap`, performs a normal transformation with `map`, and provides a single explicit fallback.

---

# 26. Anti-Pattern Conversion — Eager Database Call

### Bad

```java
User user = cache.find(id)
        .orElse(database.load(id));
```

### Better when DB loading is expensive

```java
User user = cache.find(id)
        .orElseGet(() -> database.load(id));
```

### Why?

```text
orElse()
→ database.load(id) expression evaluated eagerly

orElseGet()
→ database.load(id) invoked only when Optional is empty
```

---

# 27. Anti-Pattern Conversion — Optional Parameter

### Avoid

```java
void updatePhone(long employeeId,
                 Optional<String> phone) {
}
```

### Better design depends on the domain

```java
void updatePhone(long employeeId,
                 String phone) {
}
```

or define a request object if the operation needs to distinguish:

```text
field omitted
field explicitly cleared
field supplied with a value
```

The important point is to model those semantics explicitly rather than assuming Optional automatically solves them.

---

# 28. Optional and Null — They Can Coexist

Do not claim:

> “Good Java code never uses null.”

A more accurate statement:

> Optional can make absence explicit at selected API boundaries, but Java and existing frameworks still use null extensively. Good design is about clear contracts and controlled null handling, not blindly eliminating every null.

---

# 29. Optional vs Null — Interview Comparison

| Aspect | `null` | `Optional` |
|---|---|---|
| Communicates absence in type | No | Yes |
| Can directly call methods | Risk of NPE | Requires explicit handling |
| Good for return contract | Sometimes | Often useful |
| Good as every field | Common | Usually unnecessary |
| Good as every parameter | Common | Usually avoid |
| Supports fluent transformations | No | Yes |
| Can replace every null | No need | No |

---

# 30. Interview Questions 🔥

### Q1. Where should Optional generally be used?

Most commonly as a return type where absence is a normal, meaningful result.

### Q2. Should Optional be used as an entity field?

Usually no. Prefer the field's natural representation and expose Optional at appropriate API boundaries when useful.

### Q3. Should Optional be used as a method parameter?

Generally avoid it when a normal parameter or domain-specific request type expresses the contract more clearly.

### Q4. Why is `optional.get()` considered a code smell?

It can throw `NoSuchElementException` when the value is absent and often bypasses the explicit absence handling Optional is intended to encourage.

### Q5. Is `isPresent()` always bad?

No. It is valid when you genuinely need a presence check. The common anti-pattern is `isPresent()` immediately followed by `get()` when a clearer Optional operation exists.

### Q6. Why should an Optional-returning method never return null?

Because that violates the method's contract. Callers expect an Optional object and may themselves get an NPE if the method returns null.

### Q7. Why avoid `Optional<List<T>>`?

Usually an empty list already communicates “no elements.” But if the domain distinguishes absent collection from present-but-empty collection, Optional may be justified.

### Q8. Why avoid `List<Optional<T>>`?

Collections normally should contain actual domain values; optionality is often better represented by filtering absent results or using an appropriate domain model.

### Q9. When should `orElseGet()` be preferred?

When fallback computation is expensive, dynamic, has side effects, or should happen only when the Optional is empty.

### Q10. Does Optional eliminate null?

No. It provides an explicit abstraction for absence at selected boundaries.

### Q11. Is Optional always better than null?

No. The right choice depends on the API contract, readability, framework conventions, and domain semantics.

### Q12. Why is Optional not a universal replacement for null?

Because it can add unnecessary complexity when used for fields, parameters, collection elements, or simple local checks where ordinary control flow is clearer.

### Q13. Can Optional contain null?

No. `Optional.of(null)` throws and `Optional.ofNullable(null)` produces `Optional.empty()`.

### Q14. What is the difference between `map()` and `flatMap()` in Optional?

`map()` is for normal return values; `flatMap()` is for functions that already return Optional.

### Q15. How do you safely extract nested optional data?

Use `flatMap()` for Optional-returning methods, `map()` for normal transformations, and a suitable fallback or exception at the end.

---

# 31. Coding Challenges 🔥

### Challenge 1 — Fix `get()`

Convert:

```java
Employee e = employee.get();
```

to safe code using `orElseThrow()`.

### Challenge 2 — Remove `isPresent()` + `get()`

Rewrite:

```java
if (employee.isPresent()) {
    System.out.println(employee.get().getName());
}
```

using `ifPresent()`.

### Challenge 3 — Nested Optional

Given:

```java
Optional<Employee>
Employee::getAddress → Optional<Address>
```

retrieve the city without `get()`.

### Challenge 4 — Eager vs Lazy ⭐⭐⭐⭐⭐

Create a method that prints when called and compare:

```java
orElse(createDefault())
```

and:

```java
orElseGet(() -> createDefault())
```

with a present Optional.

### Challenge 5 — Repository API ⭐⭐⭐⭐⭐

Design:

```java
Optional<Employee> findById(long id)
```

and ensure missing employees return `Optional.empty()`, never null.

### Challenge 6 — Bad API Design

Given:

```java
void saveEmployee(Optional<Employee> employee)
```

explain why it may be unclear and design a better API.

### Challenge 7 — Complete Production Pipeline ⭐⭐⭐⭐⭐

Implement:

```text
find Employee
 → filter active employee
 → flatMap Address
 → map City
 → orElseGet default city
```

### Challenge 8 — Optional Collection

Given:

```java
List<Optional<Employee>>
```

produce a:

```java
List<Employee>
```

containing only present Employees.

---

# 32. 60-Second Interview Answer 🧠

> “I treat Optional mainly as a return-type abstraction for operations where absence is expected. I avoid using it as entity fields, method parameters, or as a wrapper around every nullable value because that can make APIs and models harder to use. I also avoid blindly calling `get()` or using `isPresent()` followed by `get()`. For composition, I use `map()` for normal transformations and `flatMap()` when the next operation returns Optional. For fallbacks, I use `orElse()` for simple values, `orElseGet()` for lazy or expensive computation, and `orElseThrow()` when absence is exceptional. The goal is not to eliminate every null; the goal is to make important absence contracts explicit and readable.”

---

# 🧠 Final Revision Sheet

```text
OPTIONAL BEST PRACTICES
────────────────────────────────────────
✓ Mainly useful as return type
✓ ofNullable() at nullable boundaries
✓ map() → normal transformation
✓ flatMap() → Optional transformation
✓ filter() → business condition
✓ orElse() → simple fallback
✓ orElseGet() → lazy/expensive fallback
✓ orElseThrow() → exceptional absence
✓ Keep pipelines readable
✓ Optional.empty(), never null Optional
```

```text
OPTIONAL ANTI-PATTERNS
────────────────────────────────────────
✗ Optional.of(null)
✗ Returning null from Optional method
✗ Blind optional.get()
✗ isPresent() + get() when unnecessary
✗ Optional fields by default
✗ Optional parameters by default
✗ Optional everywhere
✗ Unnecessary Optional<Collection<T>>
✗ Unnecessary List<Optional<T>>
✗ Nested Optional without reason
✗ Expensive fallback inside orElse()
✗ Side effects hidden inside transformations
```

### ⭐ One-Line Interview Rule

> **Use Optional to make meaningful absence explicit—not to wrap every nullable thing in Java.**

---

# 🧪 Practice Code

urlGitHub — 9.9 Optional Best Practices & Anti-Patterns Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/09-Optional-Best-Practices-Anti-Patterns

---

## Navigation

[← 9.8 — Optional `filter()` / `orElse()` / `orElseGet()](../08-Optional-filter-orElse-orElseGet/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.9 — Optional Best Practices & Anti-Patterns → ✅ Completed**

**Next → 9.10 — Optional Interview Scenarios & Final Revision**