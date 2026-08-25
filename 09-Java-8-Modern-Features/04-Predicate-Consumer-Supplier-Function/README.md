# 9.4 — Predicate / Consumer / Supplier / Function

> **Goal:** Master the four core Java 8 functional interfaces so you can choose the correct one instantly, explain their methods, composition, primitive variants, and use them in real interview scenarios.

## 🎯 The 4-Interface Mental Model

Memorize this first:

```text
Predicate<T>  → T → boolean
Consumer<T>   → T → void
Supplier<T>   → () → T
Function<T,R> → T → R
```

### One-line interview answer

> `Predicate` asks a question, `Consumer` performs an action, `Supplier` produces a value, and `Function` transforms a value.

---

# 1. `Predicate<T>` — Condition / Validation

Package:

```java
java.util.function.Predicate
```

Single abstract method:

```java
boolean test(T t);
```

Use it when the result must be `true` or `false`.

```java
Predicate<Integer> even = number -> number % 2 == 0;

System.out.println(even.test(10)); // true
System.out.println(even.test(7));  // false
```

### Real-world examples

```java
Predicate<Employee> highSalary =
        employee -> employee.getSalary() > 100000;

Predicate<String> nonEmpty =
        value -> value != null && !value.isEmpty();
```

### Memory

```text
Predicate → test() → boolean
```

---

# 2. Predicate Composition

`Predicate` provides:

```text
and()
or()
negate()
```

Example:

```java
Predicate<Integer> positive = number -> number > 0;
Predicate<Integer> even = number -> number % 2 == 0;

Predicate<Integer> positiveAndEven =
        positive.and(even);

Predicate<Integer> positiveOrEven =
        positive.or(even);

Predicate<Integer> notPositive =
        positive.negate();
```

Usage:

```java
System.out.println(positiveAndEven.test(10)); // true
System.out.println(positiveAndEven.test(-10)); // false
System.out.println(positiveOrEven.test(-10)); // true
System.out.println(notPositive.test(-5));      // true
```

### Interview trap

`and()` and `or()` use short-circuit evaluation. The second predicate may not execute when the result is already determined.

---

# 3. Predicate `isEqual()`

Java also provides:

```java
Predicate<String> isJava = Predicate.isEqual("Java");

System.out.println(isJava.test("Java")); // true
System.out.println(isJava.test("Python")); // false
```

This is useful when the condition is equality against a known target.

---

# 4. `Consumer<T>` — Action

Package:

```java
java.util.function.Consumer
```

Single abstract method:

```java
void accept(T t);
```

Use it when you consume a value and perform an action without returning a result.

```java
Consumer<String> printer =
        value -> System.out.println("Value: " + value);

printer.accept("Java");
```

### Real-world uses

```text
logging
printing
sending notification
updating an external system
persisting side effects
```

### Memory

```text
Consumer → accept() → void
```

---

# 5. Consumer Chaining

`Consumer` provides:

```java
andThen()
```

Example:

```java
Consumer<String> print =
        value -> System.out.println(value);

Consumer<String> log =
        value -> System.out.println("LOG: " + value);

Consumer<String> combined =
        print.andThen(log);

combined.accept("Java");
```

Execution order:

```text
print → log
```

### Interview point

`andThen()` executes the current consumer first and the supplied consumer second.

---

# 6. `Supplier<T>` — Producing a Value

Package:

```java
java.util.function.Supplier
```

Single abstract method:

```java
T get();
```

It takes no argument and supplies a value when `get()` is called.

```java
Supplier<String> message =
        () -> "Hello Java 8";

System.out.println(message.get());
```

### Memory

```text
Supplier → get() → value
```

---

# 7. Supplier and Lazy Evaluation

The lambda body is not executed simply because the `Supplier` is created.

```java
Supplier<String> supplier = () -> {
    System.out.println("Creating value...");
    return "Java";
};

System.out.println("Before get");
System.out.println(supplier.get());
```

Expected order:

```text
Before get
Creating value...
Java
```

This makes `Supplier` useful for deferred/lazy computation and fallback creation.

---

# 8. `Function<T,R>` — Transformation

Package:

```java
java.util.function.Function
```

Single abstract method:

```java
R apply(T t);
```

Use it when one value is transformed into another value.

```java
Function<String, Integer> length =
        value -> value.length();

System.out.println(length.apply("Java")); // 4
```

Think:

```text
T → R
```

Examples:

```java
Function<String, String> upper =
        String::toUpperCase;

Function<Employee, String> name =
        Employee::getName;
```

---

# 9. Function Composition — `andThen()`

Suppose:

```java
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
```

Then:

```java
Function<String, String> normalize =
        trim.andThen(upper);
```

Execution:

```text
input → trim → upper → output
```

Example:

```java
System.out.println(normalize.apply("  java  "));
```

Output:

```text
JAVA
```

---

# 10. Function Composition — `compose()`

```java
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;

Function<String, String> result =
        trim.compose(upper);
```

`compose()` executes the supplied function first.

General rule:

```text
f.andThen(g) → f → g
f.compose(g) → g → f
```

### ⭐ Interview question

**Q: What is the difference between `andThen()` and `compose()`?**

**Answer:** `andThen()` applies the current function first and then the supplied function. `compose()` applies the supplied function first and then the current function.

---

# 11. Function `identity()`

```java
Function<String, String> identity =
        Function.identity();

System.out.println(identity.apply("Java"));
```

It returns its input unchanged.

Conceptually:

```text
T → T
```

This is particularly useful in APIs that expect a `Function` but you want the original value.

---

# 12. Putting All Four Together ⭐⭐⭐⭐⭐

A very important interview pattern:

```java
Predicate<Employee> highSalary =
        employee -> employee.getSalary() > 80000;

Function<Employee, String> getName =
        Employee::getName;

Consumer<String> print =
        System.out::println;

Supplier<String> applicationName =
        () -> "Employee Service";
```

Think:

```text
Predicate  → should I keep it?
Function   → what should I transform it into?
Consumer   → what should I do with it?
Supplier   → where can I get/create a value?
```

---

# 13. Stream API Connection

This is one of the most important interview mappings:

```java
employees.stream()
        .filter(highSalary)
        .map(getName)
        .forEach(print);
```

Conceptually:

```text
filter()  → Predicate
map()     → Function
forEach() → Consumer
```

`Supplier` is not the standard functional interface used by `filter`, `map`, or `forEach`; it is commonly useful for producing values lazily or supplying defaults.

---

# 14. Optional Connection

These interfaces also appear in `Optional` APIs.

### `ifPresent()` → Consumer

```java
Optional<String> value = Optional.of("Java");

value.ifPresent(System.out::println);
```

### `map()` → Function

```java
value.map(String::toUpperCase);
```

### `orElseGet()` → Supplier

```java
String result = value.orElseGet(() -> "Default");
```

This is a good interview example because it connects functional interfaces to a real Java 8 API.

---

# 15. Primitive Specializations

For primitive-focused operations Java provides specialized interfaces such as:

```text
IntPredicate
IntConsumer
IntSupplier
IntFunction<R>
ToIntFunction<T>
LongPredicate
DoublePredicate
```

Example:

```java
IntPredicate positive =
        number -> number > 0;

System.out.println(positive.test(10));
```

They provide primitive-specific method signatures and can avoid boxing/unboxing where applicable.

Do not make the blanket claim that primitive specialization always guarantees a measurable performance improvement; actual performance depends on context.

---

# 16. Real-World Example — Employee Pipeline

```java
import java.util.*;
import java.util.function.*;

class Employee {

    private final int id;
    private final String name;
    private final double salary;

    Employee(int id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }
}

public class EmployeePipelineDemo {

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Amit", 50000),
                new Employee(2, "Nirbhay", 90000),
                new Employee(3, "Rahul", 120000)
        );

        Predicate<Employee> highSalary =
                employee -> employee.getSalary() > 80000;

        Function<Employee, String> getName =
                Employee::getName;

        Consumer<String> print =
                System.out::println;

        Supplier<String> serviceName =
                () -> "Employee Service";

        System.out.println(serviceName.get());

        employees.stream()
                .filter(highSalary)
                .map(getName)
                .forEach(print);
    }
}
```

Output:

```text
Employee Service
Nirbhay
Rahul
```

---

# 17. Complete Interview Practice Code

## Practice 1 — Predicate Validation

```java
import java.util.function.Predicate;

public class PredicateDemo {

    public static void main(String[] args) {

        Predicate<Integer> greaterThan50 =
                number -> number > 50;

        Predicate<Integer> even =
                number -> number % 2 == 0;

        Predicate<Integer> valid =
                greaterThan50.and(even);

        System.out.println(valid.test(60)); // true
        System.out.println(valid.test(55)); // false
    }
}
```

## Practice 2 — Consumer Chaining

```java
import java.util.function.Consumer;

public class ConsumerDemo {

    public static void main(String[] args) {

        Consumer<String> print =
                value -> System.out.println("Value: " + value);

        Consumer<String> log =
                value -> System.out.println("LOG: " + value);

        Consumer<String> combined =
                print.andThen(log);

        combined.accept("Java");
    }
}
```

## Practice 3 — Supplier / Lazy Value

```java
import java.util.function.Supplier;

public class SupplierDemo {

    public static void main(String[] args) {

        Supplier<String> supplier = () -> {
            System.out.println("Creating value...");
            return "Java 8";
        };

        System.out.println("Supplier created");
        System.out.println(supplier.get());
    }
}
```

## Practice 4 — Function Composition

```java
import java.util.function.Function;

public class FunctionDemo {

    public static void main(String[] args) {

        Function<String, String> trim = String::trim;
        Function<String, String> upper = String::toUpperCase;

        Function<String, String> normalize =
                trim.andThen(upper);

        System.out.println(normalize.apply("  nirbhay  "));
    }
}
```

## Practice 5 — Optional + Functional Interfaces

```java
import java.util.Optional;

public class OptionalFunctionalInterfaceDemo {

    public static void main(String[] args) {

        Optional<String> value = Optional.of("java");

        value.ifPresent(System.out::println);

        String upper = value
                .map(String::toUpperCase)
                .orElseGet(() -> "DEFAULT");

        System.out.println(upper);
    }
}
```

## Practice 6 — Complete Employee Interview Problem ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.function.*;

class Employee {

    private final String name;
    private final String department;
    private final double salary;

    Employee(String name, String department, double salary) {
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

    public double getSalary() {
        return salary;
    }
}

public class EmployeeFunctionalInterviewDemo {

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee("Amit", "IT", 70000),
                new Employee("Nirbhay", "IT", 120000),
                new Employee("Rahul", "HR", 90000),
                new Employee("Priya", "IT", 150000)
        );

        Predicate<Employee> isIT =
                employee -> "IT".equals(employee.getDepartment());

        Predicate<Employee> highSalary =
                employee -> employee.getSalary() > 100000;

        Predicate<Employee> eligible =
                isIT.and(highSalary);

        Function<Employee, String> nameMapper =
                Employee::getName;

        Consumer<String> printer =
                name -> System.out.println("Eligible: " + name);

        employees.stream()
                .filter(eligible)
                .map(nameMapper)
                .forEach(printer);
    }
}
```

Expected output:

```text
Eligible: Nirbhay
Eligible: Priya
```

---

# 18. `Predicate` vs `Function`

| Question | Predicate | Function |
|---|---|---|
| Input | One | One |
| Output | `boolean` | Any `R` |
| Method | `test()` | `apply()` |
| Purpose | Test | Transform |
| Stream example | `filter()` | `map()` |

### Example

```java
Predicate<String> valid =
        value -> value.length() > 5;

Function<String, Integer> length =
        String::length;
```

---

# 19. `Consumer` vs `Supplier`

| Question | Consumer | Supplier |
|---|---|---|
| Input | One | None |
| Output | `void` | `T` |
| Method | `accept()` | `get()` |
| Purpose | Action | Produce value |

Memory:

```text
Consumer → Give me a value; I'll do something.
Supplier → Ask me; I'll give you a value.
```

---

# 20. Interview Scenarios 🔥

### Scenario 1
You need to validate whether an employee is eligible.

**Answer:** `Predicate<Employee>`

### Scenario 2
You need to print/log each employee.

**Answer:** `Consumer<Employee>`

### Scenario 3
You need to generate a request ID only when needed.

**Answer:** `Supplier<String>`

### Scenario 4
You need to convert `Employee` into `EmployeeDTO`.

**Answer:** `Function<Employee, EmployeeDTO>`

### Scenario 5
You need to filter and then transform a list.

**Answer:** `Predicate` + `Function`

### Scenario 6
You need to filter, transform and finally print.

**Answer:** `Predicate` + `Function` + `Consumer`

### Scenario 7
You need a fallback that should be created only when the main value is absent.

**Answer:** Usually a `Supplier`, e.g. `Optional.orElseGet()`.

---

# 21. Common Interview Traps ❌

### Trap 1
`Predicate` returns `Boolean` as its main contract.

The method returns primitive `boolean`:

```java
boolean test(T t)
```

### Trap 2
`Consumer` can return a value.

❌ No. Its abstract method returns `void`.

### Trap 3
`Supplier` takes an argument.

❌ No. `get()` takes no arguments.

### Trap 4
`Function` always has the same input/output type.

❌ No. `Function<T,R>` can transform one type into another. Same-type transformation is what `UnaryOperator<T>` specializes.

### Trap 5
`Supplier` executes immediately when assigned.

❌ No. Its supplied computation occurs when `get()` is invoked.

### Trap 6
`andThen()` and `compose()` are identical.

❌ No. Their execution order differs.

---

# 22. Coding Challenges 🔥

Try these without opening the solutions.

### Challenge 1 — Employee Eligibility

Create:

```java
Predicate<Employee> eligible
```

Rules:

```text
age >= 18
salary >= 50000
department = IT
```

### Challenge 2 — Employee Name Transformation

Create:

```java
Function<Employee, String>
```

that returns:

```text
"ID-NAME"
```

Example:

```text
101-Nirbhay
```

### Challenge 3 — Logging Pipeline

Create a `Consumer<Employee>` that logs:

```text
Employee: Nirbhay | Salary: 100000
```

### Challenge 4 — Lazy Request ID

Create:

```java
Supplier<String>
```

that generates a UUID when `get()` is called.

### Challenge 5 — Price Pipeline ⭐⭐⭐⭐⭐

Create functions:

```text
price → discount → tax → finalPrice
```

Use `andThen()` to compose them.

### Challenge 6 — Complete Pipeline ⭐⭐⭐⭐⭐

Given a list of employees:

```text
filter IT employees
→ filter salary > 1 lakh
→ map to names
→ print names
```

Implement using only:

```text
Predicate
Function
Consumer
```

### Challenge 7 — Explain the Types

For this code, identify every functional interface:

```java
employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .forEach(System.out::println);
```

Expected:

```text
filter → Predicate<Employee>
map → Function<Employee, String>
forEach → Consumer<String>
```

---

# 23. 60-Second Interview Answer 🧠

> “The four most important Java 8 functional interfaces are Predicate, Consumer, Supplier and Function. Predicate takes one input and returns a boolean, so I use it for conditions and filtering. Consumer takes an input and returns nothing, so I use it for actions such as logging or printing. Supplier takes no input and produces a value, which is useful for lazy or deferred creation. Function takes one input and transforms it into another output. Predicate supports and, or and negate; Consumer supports andThen; Function supports compose and andThen. These interfaces are heavily used by Stream and Optional APIs.”

---

# 🧠 Final Memory Sheet

```text
┌─────────────────────────────────────────────┐
│ Predicate  → T → boolean → test()           │
│ Consumer   → T → void    → accept()         │
│ Supplier   → () → T      → get()            │
│ Function   → T → R       → apply()          │
└─────────────────────────────────────────────┘
```

Stream mapping:

```text
filter()  → Predicate
map()     → Function
forEach() → Consumer
```

Composition:

```text
Predicate → and / or / negate
Consumer  → andThen
Function  → andThen / compose / identity
Supplier  → get when value is requested
```

---

# 🧪 Practice Code

urlGitHub — 9.4 Predicate / Consumer / Supplier / Function Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/04-Predicate-Consumer-Supplier-Function

---

## Navigation

[← 9.3 — Built-in Functional Interfaces](../03-Built-in-Functional-Interfaces/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.4 — Predicate / Consumer / Supplier / Function → ✅ Completed**

**Next → 9.5 — Method References**