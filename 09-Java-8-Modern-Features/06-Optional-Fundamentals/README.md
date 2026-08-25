# 9.6 — Optional Fundamentals

> **Goal:** Understand `Optional<T>` as a Java 8 container for representing a value that may or may not be present, use it safely, and answer common interview questions around `of`, `ofNullable`, `empty`, `isPresent`, `ifPresent`, `orElse`, `orElseGet`, `orElseThrow`, `map`, and `flatMap`.

## 🎯 Interview Definition

> `Optional<T>` is a container object that may contain a non-null value or may be empty. It is primarily useful for making the absence of a return value explicit and reducing accidental null-handling errors.

Import:

```java
import java.util.Optional;
```

Basic example:

```java
Optional<String> name = Optional.of("Nirbhay");

System.out.println(name.get());
```

### Important

`Optional` does **not** make Java null-safe by itself and it does not eliminate `NullPointerException` from an application. It gives you an API for explicitly handling a potentially absent value.

---

# 1. Why `Optional`?

Traditional code:

```java
String name = getName();

if (name != null) {
    System.out.println(name.toUpperCase());
}
```

With `Optional`:

```java
Optional<String> name = getName();

name.ifPresent(value ->
        System.out.println(value.toUpperCase()));
```

The intent becomes:

```text
value present   → use it
value absent    → handle absence explicitly
```

---

# 2. Creating an Optional

There are three fundamental factory methods.

## `Optional.of()`

Use when the value is definitely non-null.

```java
Optional<String> value =
        Optional.of("Java");
```

If the argument is `null`:

```java
Optional.of(null); // ❌ NullPointerException
```

### Memory

```text
of() → value must be non-null
```

---

## `Optional.ofNullable()`

Use when the value may be null.

```java
String value = null;

Optional<String> optional =
        Optional.ofNullable(value);
```

Result:

```text
Optional.empty
```

If the value is non-null:

```java
Optional.ofNullable("Java");
```

Result:

```text
Optional[Java]
```

### Memory

```text
ofNullable() → null becomes Optional.empty()
```

---

## `Optional.empty()`

Creates an empty Optional.

```java
Optional<String> empty =
        Optional.empty();
```

It represents:

```text
No value present
```

---

# 3. `of()` vs `ofNullable()` vs `empty()` ⭐⭐⭐⭐⭐

| Method | Null allowed? | Result for null |
|---|---|---|
| `of(value)` | ❌ No | throws `NullPointerException` |
| `ofNullable(value)` | ✅ Yes | `Optional.empty()` |
| `empty()` | N/A | empty Optional |

### Interview answer

> Use `of()` when null is impossible by contract, `ofNullable()` when the source may be null, and `empty()` when you intentionally want to represent absence.

---

# 4. Checking Presence

## `isPresent()`

```java
Optional<String> value =
        Optional.of("Java");

if (value.isPresent()) {
    System.out.println(value.get());
}
```

Returns:

```text
true  → value exists
false → Optional is empty
```

## `isEmpty()`

Available since Java 11:

```java
if (value.isEmpty()) {
    System.out.println("No value");
}
```

### Interview note

For Java 8 compatibility, `isPresent()` is the relevant method. `isEmpty()` was added later.

---

# 5. `get()` — Important Trap ⚠️

```java
Optional<String> value =
        Optional.of("Java");

System.out.println(value.get());
```

If empty:

```java
Optional<String> value = Optional.empty();

value.get(); // NoSuchElementException
```

### Interview recommendation

Avoid using `get()` merely as a replacement for a null check. Prefer APIs that explicitly define what should happen when the value is absent.

---

# 6. `ifPresent()`

`ifPresent()` accepts a `Consumer`.

```java
Optional<String> value =
        Optional.of("Java");

value.ifPresent(System.out::println);
```

Conceptually:

```text
Optional<T>
     ↓
if value exists
     ↓
Consumer<T>
```

Functional-interface connection:

```java
Consumer<String> printer =
        System.out::println;
```

---

# 7. `ifPresentOrElse()`

Available since Java 9, not Java 8.

```java
Optional<String> value =
        Optional.of("Java");

value.ifPresentOrElse(
        System.out::println,
        () -> System.out.println("No value")
);
```

Do not present this as a Java 8 API in an interview.

---

# 8. `orElse()` ⭐⭐⭐⭐⭐

Provides a fallback value when the Optional is empty.

```java
Optional<String> value =
        Optional.empty();

String result =
        value.orElse("Default");

System.out.println(result);
```

Output:

```text
Default
```

### Critical interview point

The expression passed to `orElse()` is evaluated **before** the method returns, even if the Optional contains a value.

Example:

```java
String result =
        Optional.of("Java")
                .orElse(expensiveOperation());
```

`expensiveOperation()` is evaluated even though `"Java"` is present.

---

# 9. `orElseGet()` ⭐⭐⭐⭐⭐

`orElseGet()` accepts a `Supplier` and invokes it only when the Optional is empty.

```java
String result =
        Optional.of("Java")
                .orElseGet(() -> expensiveOperation());
```

If `"Java"` is present, the supplier is not invoked.

### `orElse()` vs `orElseGet()`

```text
orElse(value)
    → fallback expression is evaluated eagerly

orElseGet(supplier)
    → fallback supplier is evaluated only if needed
```

### Interview answer

> Use `orElse()` for a simple already-available fallback and `orElseGet()` when fallback creation is expensive, has side effects, or should be lazy.

---

# 10. `orElseThrow()`

Java 8:

```java
String value = Optional.of("Java")
        .orElseThrow(() ->
                new IllegalStateException("Value missing"));
```

For an empty Optional, the supplied exception is thrown.

### Java 10+

```java
String value = optional.orElseThrow();
```

This no-argument form was added later. Do not confuse it with the Java 8 API.

---

# 11. `map()` ⭐⭐⭐⭐⭐

`map()` transforms the contained value.

```java
Optional<String> name =
        Optional.of("nirbhay");

Optional<String> upper =
        name.map(String::toUpperCase);

System.out.println(upper.get());
```

Conceptually:

```text
Optional<T>
    ↓ map(Function<T,R>)
Optional<R>
```

Functional-interface connection:

```text
map() → Function
```

---

# 12. `map()` with Null Result

If the mapping function returns `null`, the resulting Optional is empty.

```java
Optional<String> result =
        Optional.of("Java")
                .map(value -> null);

System.out.println(result.isPresent()); // false
```

This is one reason `map()` is safer than manually extracting a value and then transforming it.

---

# 13. `filter()`

`filter()` keeps the value only if a predicate matches.

```java
Optional<Integer> number =
        Optional.of(100);

Optional<Integer> result =
        number.filter(value -> value > 50);
```

Conceptually:

```text
filter() → Predicate
```

If the predicate returns `false`, the result is empty.

---

# 14. `flatMap()` ⭐⭐⭐⭐⭐

`flatMap()` is useful when the mapping function already returns an `Optional`.

Suppose:

```java
Optional<String> findDepartment(Employee employee)
```

Then:

```java
Optional<Department> department =
        employeeOptional.flatMap(this::findDepartment);
```

Why not `map()`?

Because `map()` would create nested structure:

```text
Optional<Optional<Department>>
```

`flatMap()` flattens it to:

```text
Optional<Department>
```

### Memory

```text
map     → T → R
flatMap → T → Optional<R>
```

---

# 15. `map()` vs `flatMap()` 🔥

### `map()`

```java
Optional<String> name =
        Optional.of("Java");

Optional<Integer> length =
        name.map(String::length);
```

Mapper returns a normal value:

```text
String → Integer
```

### `flatMap()`

```java
Optional<Address> address =
        employee.flatMap(Employee::getAddress);
```

Mapper already returns Optional:

```text
Employee → Optional<Address>
```

### Interview answer

> `map()` wraps a normal mapping result in an Optional, while `flatMap()` is used when the mapping function already returns an Optional and prevents nested Optionals.

---

# 16. Optional + Functional Interfaces

This connects directly with 9.4.

```text
ifPresent()  → Consumer
map()        → Function
filter()     → Predicate
orElseGet()  → Supplier
```

Example:

```java
Optional<String> name =
        Optional.of("nirbhay");

name.filter(value -> value.length() > 3)
        .map(String::toUpperCase)
        .ifPresent(System.out::println);
```

Flow:

```text
Optional<String>
      ↓
filter → Predicate
      ↓
map → Function
      ↓
ifPresent → Consumer
```

---

# 17. Complete Employee Example ⭐⭐⭐⭐⭐

```java
import java.util.*;

class Employee {

    private final int id;
    private final String name;
    private final String department;

    Employee(int id, String name, String department) {
        this.id = id;
        this.name = name;
        this.department = department;
    }

    public String getName() {
        return name;
    }

    public String getDepartment() {
        return department;
    }
}

public class EmployeeOptionalDemo {

    public static Optional<Employee> findEmployee(
            List<Employee> employees, int id) {

        return employees.stream()
                .filter(employee -> employee.id == id)
                .findFirst();
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Amit", "HR"),
                new Employee(2, "Nirbhay", "IT"),
                new Employee(3, "Rahul", "Finance")
        );

        Optional<Employee> employee =
                findEmployee(employees, 2);

        employee
                .map(Employee::getName)
                .map(String::toUpperCase)
                .ifPresent(System.out::println);
    }
}
```

Expected output:

```text
NIRBHAY
```

### Interview improvement

In production code, expose `Optional<Employee>` from a method when absence is a meaningful part of the API contract. Do not mechanically wrap every field or every local variable in `Optional`.

---

# 18. Optional Return Type ⭐⭐⭐⭐⭐

Good use:

```java
public Optional<Employee> findById(int id) {
    // return employee if found
}
```

Caller:

```java
employeeRepository.findById(101)
        .map(Employee::getName)
        .ifPresent(System.out::println);
```

This makes absence explicit in the method contract.

---

# 19. Optional Should Usually Not Be Used as a Field

Avoid casually designing entities like:

```java
class Employee {
    private Optional<String> name;
}
```

`Optional` was primarily designed for return types and fluent absence handling, not as a general-purpose replacement for every nullable field.

In particular, many frameworks and serialization libraries have their own expectations around fields, constructors, persistence and serialization.

### Interview answer

> `Optional` is generally most useful at API boundaries and return types where absence is meaningful, rather than as a universal replacement for null or as a field type everywhere.

---

# 20. Optional as a Method Parameter?

Avoid using it routinely:

```java
void process(Optional<String> name) // generally avoid
```

Prefer a clear API contract, for example:

```java
void process(String name)
```

or, if null is genuinely part of the contract, document and validate it appropriately.

There are exceptions based on API design, but the general interview recommendation is not to use `Optional` indiscriminately as parameters.

---

# 21. Optional Primitive Variants

Java provides:

```text
OptionalInt
OptionalLong
OptionalDouble
```

Example:

```java
OptionalInt value =
        OptionalInt.of(100);

System.out.println(value.getAsInt());
```

These avoid wrapping the primitive value in `Optional<Integer>`, etc.

---

# 22. `Optional` Does Not Mean “Never Null”

This is an important design point.

If a method declares:

```java
Optional<String> getName()
```

the method should normally return:

```java
Optional.of(value)
```

or:

```java
Optional.empty()
```

not:

```java
return null; // ❌ defeats the contract
```

The caller expects an Optional object and should not have to perform another null check on the Optional itself.

---

# 23. Common Mistake — `isPresent()` + `get()`

This is technically safe:

```java
if (optional.isPresent()) {
    String value = optional.get();
}
```

But often less expressive than:

```java
optional.ifPresent(value -> {
    // use value
});
```

Or:

```java
String value = optional.orElse("Default");
```

The better API depends on what you actually want to do.

---

# 24. `orElse()` vs `orElseGet()` — Practice ⭐⭐⭐⭐⭐

```java
public static String createDefault() {
    System.out.println("Creating default...");
    return "DEFAULT";
}

public static void main(String[] args) {

    Optional<String> present =
            Optional.of("JAVA");

    String a = present.orElse(createDefault());

    String b = present.orElseGet(() -> createDefault());

    System.out.println(a);
    System.out.println(b);
}
```

The key observation:

```text
orElse()    → createDefault() is evaluated
orElseGet() → supplier is not invoked when value exists
```

This is a classic interview question.

---

# 25. Complete Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.Optional;

public class OptionalFundamentalsPractice {

    static String getName(boolean present) {
        return present ? "Nirbhay" : null;
    }

    static String expensiveDefault() {
        System.out.println("Creating expensive default...");
        return "Default User";
    }

    public static void main(String[] args) {

        // 1. of()
        Optional<String> name =
                Optional.of("Nirbhay");

        // 2. ofNullable()
        Optional<String> nullableName =
                Optional.ofNullable(getName(false));

        // 3. empty()
        Optional<String> empty =
                Optional.empty();

        // 4. Presence
        System.out.println(name.isPresent());
        System.out.println(empty.isPresent());

        // 5. ifPresent()
        name.ifPresent(System.out::println);

        // 6. map()
        Optional<String> upper =
                name.map(String::toUpperCase);

        System.out.println(upper.orElse("NO NAME"));

        // 7. filter()
        Optional<String> filtered =
                name.filter(value -> value.length() > 5);

        System.out.println(filtered.orElse("Too Short"));

        // 8. orElse()
        String value1 =
                empty.orElse("DEFAULT");

        // 9. orElseGet()
        String value2 =
                name.orElseGet(() -> expensiveDefault());

        System.out.println(value1);
        System.out.println(value2);

        // 10. orElseThrow()
        String value3 =
                name.orElseThrow(() ->
                        new IllegalStateException("Name missing"));

        System.out.println(value3);
    }
}
```

---

# 26. Interview Scenarios 🔥

### Scenario 1 — Repository lookup

```java
Optional<Employee> findById(int id)
```

**Why?** Employee may not exist.

### Scenario 2 — Transform optional employee

```java
employee.map(Employee::getName)
```

**Why?** `map()` uses a `Function` to transform the contained value.

### Scenario 3 — Execute only when present

```java
employee.ifPresent(System.out::println);
```

**Why?** `ifPresent()` accepts a `Consumer`.

### Scenario 4 — Lazy fallback

```java
employee.orElseGet(this::createDefaultEmployee)
```

**Why?** `orElseGet()` uses a `Supplier` and evaluates it only when needed.

### Scenario 5 — Validate contained value

```java
employee.filter(e -> e.getSalary() > 100000)
```

**Why?** `filter()` uses a `Predicate`.

---

# 27. Interview Questions 🔥

### Q1. What is Optional?

A container representing either a non-null value or absence of a value.

### Q2. Why was Optional introduced?

To make absence explicit in APIs and provide fluent operations for handling potentially missing values.

### Q3. `of()` vs `ofNullable()`?

`of()` rejects null; `ofNullable()` converts null to an empty Optional.

### Q4. What happens when `get()` is called on an empty Optional?

`NoSuchElementException`.

### Q5. `orElse()` vs `orElseGet()`?

`orElse()` receives an eagerly evaluated fallback expression; `orElseGet()` receives a `Supplier` and evaluates it only when the Optional is empty.

### Q6. `map()` vs `flatMap()`?

`map()` is for a mapper returning a normal value; `flatMap()` is for a mapper returning an Optional and avoids nested Optionals.

### Q7. Which functional interface does `map()` use?

`Function`.

### Q8. Which functional interface does `filter()` use?

`Predicate`.

### Q9. Which functional interface does `ifPresent()` use?

`Consumer`.

### Q10. Which functional interface does `orElseGet()` use?

`Supplier`.

### Q11. Can Optional itself be null?

It technically can if a method incorrectly returns null, but an API returning `Optional` should normally return an actual Optional object—either present or empty.

### Q12. Should Optional be used for every field?

No. It is generally intended for expressing optional return values and fluent absence handling, not as a universal replacement for null.

### Q13. Is Optional a replacement for all null checks?

No. It is an API for representing and handling absence; it does not make arbitrary Java code null-safe.

### Q14. Is `Optional` available for primitives?

Yes: `OptionalInt`, `OptionalLong`, and `OptionalDouble`.

### Q15. Is `ifPresentOrElse()` Java 8?

No. It was added in Java 9.

---

# 28. Common Interview Traps ❌

### Trap 1

```java
Optional.of(null)
```

❌ Throws `NullPointerException`.

### Trap 2

```java
optional.get()
```

❌ Not safe when empty; throws `NoSuchElementException`.

### Trap 3

`orElse()` and `orElseGet()` have identical evaluation behavior.

❌ They differ in eager vs lazy fallback evaluation.

### Trap 4

`Optional` completely eliminates null.

❌ No.

### Trap 5

`map()` and `flatMap()` are identical.

❌ `flatMap()` prevents nested `Optional` results.

### Trap 6

`Optional` should replace every nullable field.

❌ Not recommended as a blanket rule.

### Trap 7

`ifPresentOrElse()` is Java 8.

❌ It is Java 9+.

---

# 29. Coding Challenges 🔥

### Challenge 1 — Safe Name Lookup

Create:

```java
Optional<String> findName(Integer id)
```

Return a name when found and `Optional.empty()` otherwise.

### Challenge 2 — Transform Name

Given:

```java
Optional<String> name
```

convert it to uppercase using `map()`.

### Challenge 3 — Salary Validation

Given:

```java
Optional<Employee> employee
```

keep the employee only when salary is greater than `100000` using `filter()`.

### Challenge 4 — Lazy Default

Use `orElseGet()` to create a default employee only when the employee is absent.

### Challenge 5 — `map()` vs `flatMap()` ⭐⭐⭐⭐⭐

Create:

```java
Optional<Employee>
Optional<Address>
```

and demonstrate why a method returning `Optional<Address>` should be composed with `flatMap()`.

### Challenge 6 — Complete Pipeline ⭐⭐⭐⭐⭐

Implement:

```text
find employee
→ filter salary > 1 lakh
→ get name
→ uppercase
→ print if present
```

Use only Optional operations.

### Challenge 7 — `orElse` Trap ⭐⭐⭐⭐⭐

Write code that demonstrates the difference between:

```java
orElse()
orElseGet()
```

using a method that prints when it is invoked.

---

# 30. 60-Second Interview Answer 🧠

> “Optional is a Java 8 container that can represent either a non-null value or an empty state. It is useful mainly for making absence explicit in APIs and handling it fluently. `of()` is used for a definitely non-null value, while `ofNullable()` converts null into an empty Optional. `map()` transforms a value, `filter()` applies a condition, `ifPresent()` performs an action, `orElse()` provides an eager fallback, and `orElseGet()` provides a lazy Supplier-based fallback. `flatMap()` is useful when the mapping function already returns an Optional. I would not use Optional as a universal replacement for null or automatically wrap every field or parameter.”

---

# 🧠 Final Memory Sheet

```text
CREATE
────────────────────────────
of()          → non-null only
ofNullable()  → null → empty
empty()       → no value

CHECK
────────────────────────────
isPresent()   → value exists?
isEmpty()     → empty? (Java 11+)

USE
────────────────────────────
get()         → value / exception if empty
ifPresent()   → Consumer

FALLBACK
────────────────────────────
orElse()      → eager fallback
orElseGet()   → lazy Supplier
orElseThrow() → throw if empty

TRANSFORM
────────────────────────────
map()         → Function
flatMap()     → Function returning Optional
filter()      → Predicate
```

### ⭐ Functional Interface Connection

```text
Optional.map()       → Function
Optional.filter()    → Predicate
Optional.ifPresent() → Consumer
Optional.orElseGet() → Supplier
```

---

# 🧪 Practice Code

urlGitHub — 9.6 Optional Fundamentals Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/06-Optional-Fundamentals

---

## Navigation

[← 9.5 — Method References](../05-Method-References/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.6 — Optional Fundamentals → ✅ Completed**

**Next → 9.7 — Optional `map()` / `flatMap()`**