# 9.8 — Optional `filter()` / `orElse()` / `orElseGet()`

> **Interview Goal:** Understand conditional filtering and fallback strategies in `Optional`, especially the most common interview trap: **`orElse()` is eager while `orElseGet()` is lazy**.

## 🎯 30-Second Interview Definition

> `filter()` keeps an Optional value only when a `Predicate` matches. `orElse()` returns a fallback value when the Optional is empty but evaluates its argument eagerly. `orElseGet()` accepts a `Supplier` and creates the fallback lazily, only when the Optional is empty.

---

# 1. `filter()` Fundamentals

`filter()` accepts a `Predicate`:

```java
Optional<Integer> salary = Optional.of(150000);

Optional<Integer> result =
        salary.filter(value -> value > 100000);
```

If the predicate returns `true`:

```text
Optional[150000]
```

If it returns `false`:

```text
Optional.empty
```

### Memory

```text
filter() → Predicate → keep or discard
```

---

# 2. `filter()` Signature

Conceptually:

```java
Optional<T> filter(Predicate<? super T> predicate)
```

It does **not** transform the value's type.

```text
Optional<T>
    ↓ Predicate<T>
Optional<T>
```

Compare with `map()`:

```text
map()    → Optional<T> → Optional<R>
filter() → Optional<T> → Optional<T>
```

---

# 3. `filter()` with Employee

```java
Optional<Employee> employee =
        Optional.of(new Employee("Nirbhay", 120000));

Optional<Employee> eligible =
        employee.filter(e -> e.getSalary() > 100000);
```

If salary is `120000`, the Employee remains present.

If salary is `80000`, the result becomes empty.

---

# 4. `filter()` on an Empty Optional

```java
Optional<Integer> empty = Optional.empty();

Optional<Integer> result =
        empty.filter(value -> value > 100);
```

The predicate is not invoked because there is no value.

Result:

```text
Optional.empty
```

---

# 5. `filter()` + `map()` ⭐⭐⭐⭐⭐

A very common pipeline:

```java
String name = employee
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .orElse("Not Eligible");
```

Flow:

```text
Optional<Employee>
       ↓
filter → Predicate
       ↓
Optional<Employee>
       ↓
map → Function
       ↓
Optional<String>
       ↓
orElse → fallback
       ↓
String
```

---

# 6. `filter()` vs `map()`

| Operation | Purpose | Example |
|---|---|---|
| `filter()` | Keep/discard value | `salary > 100000` |
| `map()` | Transform value | `Employee → String` |
| `flatMap()` | Compose Optional-returning transformation | `Employee → Optional<Address>` |

### Golden rule

```text
Need condition?       → filter()
Need transformation?  → map()
Returns Optional?     → flatMap()
```

---

# 7. `orElse()` Fundamentals

`orElse()` provides a fallback value:

```java
Optional<String> name = Optional.empty();

String result =
        name.orElse("Unknown");

System.out.println(result);
```

Output:

```text
Unknown
```

If the Optional contains a value:

```java
Optional<String> name = Optional.of("Nirbhay");

String result =
        name.orElse("Unknown");
```

Result:

```text
Nirbhay
```

---

# 8. `orElse()` Is Eager ⭐⭐⭐⭐⭐

This is one of the most frequently asked Optional questions.

```java
static String createDefault() {
    System.out.println("Creating default...");
    return "DEFAULT";
}

Optional<String> value =
        Optional.of("JAVA");

String result =
        value.orElse(createDefault());
```

Even though `value` is present, `createDefault()` is evaluated before `orElse()` is invoked.

Output includes:

```text
Creating default...
```

### Why?

Java evaluates method arguments before calling the method.

Conceptually:

```java
String fallback = createDefault();
String result = value.orElse(fallback);
```

So `orElse()` is **eager**.

---

# 9. `orElseGet()` Fundamentals

`orElseGet()` accepts a `Supplier`:

```java
String result =
        value.orElseGet(() -> createDefault());
```

The supplier is invoked only if the Optional is empty.

If:

```java
Optional<String> value = Optional.of("JAVA");
```

then `createDefault()` is not executed.

### Memory

```text
orElse()    → value
orElseGet() → Supplier
```

---

# 10. `orElse()` vs `orElseGet()` ⭐⭐⭐⭐⭐

| | `orElse()` | `orElseGet()` |
|---|---|---|
| Parameter | Value | `Supplier` |
| Evaluation | Eager | Lazy |
| Fallback created when present? | Yes, expression is evaluated | No |
| Best for | Simple/cheap fallback | Expensive/dynamic fallback |

### Interview answer

> `orElse()` evaluates the fallback expression regardless of whether the Optional contains a value. `orElseGet()` receives a Supplier and invokes it only when the Optional is empty.

---

# 11. The Classic Interview Output Question 🔥

```java
static String defaultValue() {
    System.out.println("defaultValue called");
    return "DEFAULT";
}

public static void main(String[] args) {

    Optional<String> value =
            Optional.of("JAVA");

    String a = value.orElse(defaultValue());

    String b = value.orElseGet(() -> defaultValue());
}
```

What happens?

For `orElse()`:

```text
defaultValue called
```

For `orElseGet()`:

```text
nothing is printed
```

because the value is already present.

---

# 12. When Optional Is Empty

```java
Optional<String> value = Optional.empty();

String a = value.orElse(defaultValue());
String b = value.orElseGet(() -> defaultValue());
```

Both need the fallback, so both invoke the fallback creation.

Conceptually:

```text
empty Optional
     ↓
 fallback required
     ↓
 both execute their fallback
```

---

# 13. `orElse()` with an Expensive Operation

Avoid unnecessarily doing:

```java
Employee employee =
        employeeOptional.orElse(createEmployeeFromDatabase());
```

If `employeeOptional` already contains an Employee, the database-related expression may still be evaluated.

Prefer lazy evaluation when creation is expensive:

```java
Employee employee =
        employeeOptional.orElseGet(
                () -> createEmployeeFromDatabase());
```

### Production rule

```text
Cheap already-created fallback → orElse()
Expensive/dynamic fallback       → orElseGet()
```

---

# 14. `orElseGet()` and Supplier

Because:

```java
orElseGet(Supplier<? extends T> supplier)
```

it connects directly to Java 8 functional interfaces.

Example:

```java
Supplier<String> supplier =
        () -> "DEFAULT";

String result =
        Optional.<String>empty()
                .orElseGet(supplier);
```

Memory:

```text
Supplier → produces a value
orElseGet → asks Supplier for fallback only if needed
```

---

# 15. `orElse()` with `null`

This is legal:

```java
String result =
        optional.orElse(null);
```

But it defeats much of the benefit of expressing absence through Optional and should generally not be used as a way to immediately convert back to null unless the surrounding API explicitly requires null.

---

# 16. `orElse()` vs `orElseThrow()`

Use `orElse()` when a valid fallback exists:

```java
String name =
        optionalName.orElse("Unknown");
```

Use `orElseThrow()` when absence is exceptional:

```java
Employee employee =
        employeeOptional.orElseThrow(
                () -> new EmployeeNotFoundException("Employee not found"));
```

### Decision

```text
Missing is normal      → fallback
Missing is exceptional → throw
```

---

# 17. Complete Employee Example ⭐⭐⭐⭐⭐

```java
import java.util.Optional;

class Employee {

    private final int id;
    private final String name;
    private final double salary;

    Employee(int id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }

    @Override
    public String toString() {
        return id + " - " + name + " - " + salary;
    }
}

public class OptionalFilterFallbackDemo {

    static Employee createDefaultEmployee() {
        System.out.println("Creating default employee...");
        return new Employee(0, "Default", 0);
    }

    public static void main(String[] args) {

        Optional<Employee> employee =
                Optional.of(
                        new Employee(101, "Nirbhay", 150000));

        String eligibleName = employee
                .filter(e -> e.getSalary() > 100000)
                .map(Employee::getName)
                .orElse("Not Eligible");

        System.out.println(eligibleName);

        Employee result = employee
                .orElseGet(OptionalFilterFallbackDemo::createDefaultEmployee);

        System.out.println(result);
    }
}
```

Expected output:

```text
Nirbhay
101 - Nirbhay - 150000.0
```

Notice that the default employee is not created because the Optional is present.

---

# 18. Complete Interview Practice Program 🔥

```java
import java.util.Optional;
import java.util.function.Supplier;

public class OptionalFilterOrElsePractice {

    static String createDefault() {
        System.out.println("Creating default value...");
        return "DEFAULT";
    }

    static String findUserName(int id) {
        if (id == 101) {
            return "Nirbhay";
        }
        return null;
    }

    public static void main(String[] args) {

        // 1. Create Optional from nullable source
        Optional<String> name =
                Optional.ofNullable(findUserName(101));

        // 2. filter()
        Optional<String> validName =
                name.filter(value -> value.length() >= 5);

        System.out.println(
                "Filtered: " + validName.orElse("Invalid Name"));

        // 3. map() + filter() + orElse()
        String upperName = name
                .filter(value -> value.length() >= 5)
                .map(String::toUpperCase)
                .orElse("UNKNOWN");

        System.out.println("Uppercase: " + upperName);

        // 4. orElse() - eager
        String eager = name.orElse(createDefault());
        System.out.println("Eager result: " + eager);

        // 5. orElseGet() - lazy
        String lazy = name.orElseGet(() -> createDefault());
        System.out.println("Lazy result: " + lazy);

        // 6. Explicit Supplier
        Supplier<String> supplier =
                OptionalFilterOrElsePractice::createDefault;

        String supplierResult =
                Optional.<String>empty()
                        .orElseGet(supplier);

        System.out.println(
                "Supplier result: " + supplierResult);
    }
}
```

### Expected key observation

For the present Optional, `createDefault()` is called by `orElse()` but not by `orElseGet()`.

---

# 19. Practice: Missing User

Change:

```java
Optional<String> name =
        Optional.ofNullable(findUserName(999));
```

Now test:

```java
name.filter(value -> value.length() >= 5)
        .map(String::toUpperCase)
        .orElse("UNKNOWN");
```

Result:

```text
UNKNOWN
```

Then compare:

```java
name.orElse(createDefault());
```

with:

```java
name.orElseGet(() -> createDefault());
```

Both need the fallback because the Optional is empty.

---

# 20. Chaining `filter()` Multiple Times

You can apply multiple predicates:

```java
Optional<Integer> salary = Optional.of(150000);

Optional<Integer> result = salary
        .filter(value -> value > 100000)
        .filter(value -> value < 200000);
```

Both conditions must pass.

Conceptually:

```text
150000
  ↓ >100000   ✓
150000
  ↓ <200000   ✓
150000
```

If either condition fails:

```text
Optional.empty()
```

---

# 21. `filter()` Is Short-Circuiting in the Pipeline

Once the Optional becomes empty, later Optional operations do not receive a value.

```java
Optional<Integer> result =
        Optional.of(50)
                .filter(value -> value > 100)
                .filter(value -> {
                    System.out.println("Second filter");
                    return true;
                });
```

The second predicate is not executed because the first filter already produced an empty Optional.

---

# 22. Real Interview Scenario — Eligibility ⭐⭐⭐⭐⭐

Question:

> You have an employee Optional. Return the employee's name only when salary is greater than 10 LPA. Otherwise return `"Not Eligible"`.

Answer:

```java
String result = employeeOptional
        .filter(employee -> employee.getSalary() > 100000)
        .map(Employee::getName)
        .orElse("Not Eligible");
```

Explain:

```text
filter → business condition
map    → extract/transform name
orElse → fallback
```

---

# 23. Real Interview Scenario — Expensive Fallback ⭐⭐⭐⭐⭐

Question:

> You have a cached user. If it exists, return it. Otherwise load it from the database. Which method do you choose?

Preferred:

```java
User user = cachedUser
        .orElseGet(() -> userRepository.loadFromDatabase(id));
```

Why?

Because database access should happen only when the cache value is absent.

Avoid:

```java
User user = cachedUser
        .orElse(userRepository.loadFromDatabase(id));
```

when the fallback operation is expensive or has side effects, because the argument is evaluated eagerly.

---

# 24. Important Nuance — `orElse()` Does Not Mean “Always Returns Fallback”

This is wrong:

> `orElse()` always executes and returns the default.

Correct:

> The fallback expression is evaluated eagerly, but the fallback value is returned only when the Optional is empty.

Example:

```java
Optional<String> value = Optional.of("JAVA");

String result = value.orElse(createDefault());
```

`createDefault()` executes, but:

```text
result = JAVA
```

not `DEFAULT`.

This distinction is important in interviews.

---

# 25. Important Nuance — `orElseGet()` Is Not “Always Lazy” in the Broad Sense

`orElseGet()` delays invocation of the **Supplier's `get()`** until the Optional is empty.

For example:

```java
Supplier<String> supplier = createSupplier();

optional.orElseGet(supplier);
```

`createSupplier()` itself is executed when the Supplier object is created.

The lazy part is the Supplier's fallback computation:

```java
supplier.get()
```

when needed.

---

# 26. `filter()` with `map()` and `flatMap()`

A realistic chain can combine all three:

```java
String city = employeeOptional
        .filter(e -> e.getSalary() > 100000)
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .orElseGet(() -> "Unknown City");
```

Flow:

```text
Optional<Employee>
       ↓
filter    → Predicate
       ↓
flatMap   → Optional<Address>
       ↓
map       → String
       ↓
orElseGet → lazy fallback
       ↓
String
```

This is excellent interview-level code because it combines the previous Optional topics.

---

# 27. Common Mistakes ❌

### Mistake 1

```java
optional.filter(x -> x > 10).get();
```

If the filter fails, `get()` throws `NoSuchElementException`.

Prefer a meaningful fallback or exception policy.

### Mistake 2

Using `orElse()` for an expensive database/API call:

```java
optional.orElse(callDatabase());
```

Prefer `orElseGet()` when the call should happen only when absent.

### Mistake 3

Thinking `filter()` changes the value.

It does not transform the value; it either retains it or produces an empty Optional.

### Mistake 4

Thinking `orElse()` returns the fallback even when a value exists.

It doesn't. The fallback expression is evaluated, but the existing Optional value is returned.

### Mistake 5

Using `orElseGet()` for a trivial already-created constant without a reason.

```java
optional.orElseGet(() -> "UNKNOWN");
```

This is valid but adds unnecessary indirection for a simple constant. `orElse("UNKNOWN")` is clearer.

---

# 28. Interview Questions 🔥

### Q1. What does `filter()` do?

It keeps the Optional value only when a supplied Predicate returns true.

### Q2. Does `filter()` transform the value?

No. It keeps the same type and either preserves the value or returns empty.

### Q3. What happens if the Optional is already empty before `filter()`?

The predicate is not executed and the result remains empty.

### Q4. What is the biggest difference between `orElse()` and `orElseGet()`?

Eager vs lazy fallback evaluation.

### Q5. Which functional interface does `orElseGet()` use?

`Supplier`.

### Q6. Why can `orElse()` be a performance problem?

An expensive fallback expression is evaluated even when the Optional contains a value.

### Q7. Does `orElse()` return the fallback when the Optional is present?

No. It returns the contained value, but the fallback expression has already been evaluated.

### Q8. Give a practical `orElseGet()` use case.

Loading a value from a database or remote service only when a cached Optional is empty.

### Q9. How do `filter()`, `map()`, and `flatMap()` differ?

`filter()` keeps/discards, `map()` transforms to a normal value, and `flatMap()` composes an Optional-returning transformation.

### Q10. Can multiple `filter()` calls be chained?

Yes. Each predicate must pass for the value to remain present.

### Q11. What happens when a filter predicate returns false?

The result becomes `Optional.empty()`.

### Q12. Is `orElseGet()` always better than `orElse()`?

No. Use `orElse()` for simple/cheap fallback values and `orElseGet()` when lazy computation matters.

### Q13. What is the return type of `filter()`?

`Optional<T>`.

### Q14. Why is `orElse()` called eager?

Because its argument is evaluated before the method is invoked.

### Q15. Why is `orElseGet()` called lazy?

Because the Supplier's `get()` is invoked only when the Optional is empty.

---

# 29. Coding Challenges 🔥

### Challenge 1 — Salary Filter

Given:

```java
Optional<Employee> employee
```

keep the employee only when salary is greater than `100000`.

### Challenge 2 — Eligibility Pipeline ⭐⭐⭐⭐⭐

Implement:

```text
Employee
 → salary > 10 LPA
 → get name
 → uppercase
 → fallback "NOT ELIGIBLE"
```

### Challenge 3 — Multiple Filters

Keep a number only when:

```text
> 50
< 200
is even
```

Use chained `filter()` calls.

### Challenge 4 — Eager vs Lazy ⭐⭐⭐⭐⭐

Create a `createDefault()` method that prints when called. Compare:

```java
orElse(createDefault())
```

with:

```java
orElseGet(() -> createDefault())
```

using a present Optional.

### Challenge 5 — Cache + Database ⭐⭐⭐⭐⭐

Given:

```java
Optional<User> cachedUser
```

load from database only when absent.

### Challenge 6 — Complete Optional Pipeline ⭐⭐⭐⭐⭐

Implement:

```text
Optional<Employee>
 → filter salary
 → flatMap address
 → map city
 → orElseGet default city
```

### Challenge 7 — Predict Output

Without running this:

```java
Optional<String> value = Optional.of("Java");

String result = value.orElse(printAndReturnDefault());
```

Answer:

```text
printAndReturnDefault() executes
result is "Java"
```

---

# 30. 60-Second Interview Answer 🧠

> “`Optional.filter()` takes a Predicate and keeps the value only if the condition passes. It doesn't transform the value. `orElse()` provides a fallback, but its argument is eagerly evaluated because Java evaluates method arguments before invocation. `orElseGet()` takes a Supplier, so the fallback computation happens only when the Optional is empty. For example, if I have a cached user and need to hit the database only when the cache is empty, I would use `orElseGet()`. In a typical pipeline I might use `filter()` for business eligibility, `map()` to transform the object, and `orElseGet()` for an expensive fallback.”

---

# 🧠 Final Revision Sheet

```text
filter()
──────────────
Predicate
↓
keep / discard
↓
Optional<T>
```

```text
orElse()
──────────────
Fallback value
↓
EAGER evaluation
```

```text
orElseGet()
──────────────
Supplier
↓
LAZY evaluation
↓
only when Optional is empty
```

### ⭐ Golden Rules

```text
Condition?             → filter()
Transform?             → map()
Returns Optional?      → flatMap()
Simple fallback?       → orElse()
Expensive/dynamic?     → orElseGet()
Missing is exceptional → orElseThrow()
```

### ⭐ Functional Interface Mapping

```text
filter()    → Predicate
map()       → Function
flatMap()   → Function returning Optional
orElseGet() → Supplier
ifPresent() → Consumer
```

### ⭐ Best Interview Example

```java
String city = employeeOptional
        .filter(e -> e.getSalary() > 100000)
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .orElseGet(() -> "Unknown City");
```

---

# 🧪 Practice Code

urlGitHub — 9.8 Optional `filter()` / `orElse()` / `orElseGet()` Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/08-Optional-filter-orElse-orElseGet

---

## Navigation

[← 9.7 — Optional `map()` / `flatMap()`](../07-Optional-map-flatMap/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.8 — Optional `filter()` / `orElse()` / `orElseGet()` → ✅ Completed**

**Next → 9.9 — Optional Best Practices & Anti-Patterns**