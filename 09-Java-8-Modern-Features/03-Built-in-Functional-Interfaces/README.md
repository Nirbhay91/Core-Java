# 9.3 — Built-in Functional Interfaces

> **Goal:** Master Java 8's most-used functional interfaces and know exactly when to use each one in interviews and production code.

## 🎯 Interview Definition

> Java provides ready-made functional interfaces in `java.util.function` so developers do not need to create a custom interface for common patterns such as validation, consuming a value, supplying a value, transforming a value, or combining two values.

Core interfaces:

```text
Predicate<T>      → T → boolean
Consumer<T>       → T → void
Supplier<T>       → () → T
Function<T,R>     → T → R

UnaryOperator<T>  → T → T
BinaryOperator<T> → (T,T) → T

BiPredicate<T,U>  → (T,U) → boolean
BiConsumer<T,U>   → (T,U) → void
BiFunction<T,U,R> → (T,U) → R
```

---

# 1. `Predicate<T>`

### Purpose

Use `Predicate` when you need to **test a condition**.

Abstract method:

```java
boolean test(T t);
```

Example:

```java
Predicate<Integer> isEven = number -> number % 2 == 0;

System.out.println(isEven.test(10)); // true
System.out.println(isEven.test(7));  // false
```

### Memory trick

```text
Predicate → P → Predicate/Problem condition → boolean
```

Think:

```text
Input → condition → true/false
```

### Real-world example

```java
Predicate<Employee> highSalary =
        employee -> employee.getSalary() > 100000;
```

---

# 2. `Consumer<T>`

### Purpose

Use `Consumer` when you receive a value and perform an action but **return nothing**.

Abstract method:

```java
void accept(T t);
```

Example:

```java
Consumer<String> printer =
        name -> System.out.println("Hello " + name);

printer.accept("Nirbhay");
```

### Memory trick

```text
Consumer → consumes input → no return
```

Common uses:

- logging
- printing
- sending notifications
- updating state
- `forEach()`

---

# 3. `Supplier<T>`

### Purpose

Use `Supplier` when there is **no input but a value is produced**.

Abstract method:

```java
T get();
```

Example:

```java
Supplier<String> message =
        () -> "Hello Java 8";

System.out.println(message.get());
```

Another example:

```java
Supplier<Double> randomNumber = Math::random;
```

### Memory trick

```text
Supplier → supplies a value → no input
```

Common uses:

- lazy creation
- fallback values
- factories
- deferred computation

---

# 4. `Function<T,R>`

### Purpose

Use `Function` when you need to **transform one value into another value**.

Abstract method:

```java
R apply(T t);
```

Example:

```java
Function<String, Integer> length =
        text -> text.length();

System.out.println(length.apply("Java")); // 4
```

Think:

```text
T → R
```

Real-world example:

```java
Function<Employee, String> employeeName =
        Employee::getName;
```

---

# 5. `UnaryOperator<T>`

`UnaryOperator<T>` is a specialized `Function<T,T>`.

```text
One input → same type output
```

Example:

```java
UnaryOperator<Integer> square =
        number -> number * number;

System.out.println(square.apply(5)); // 25
```

Relationship:

```text
UnaryOperator<T>
       ↓
extends
Function<T,T>
```

### Interview line

> Use `UnaryOperator<T>` when the input and output are the same type.

---

# 6. `BinaryOperator<T>`

`BinaryOperator<T>` is a specialized `BiFunction<T,T,T>`.

```text
T + T → T
```

Example:

```java
BinaryOperator<Integer> add =
        (a, b) -> a + b;

System.out.println(add.apply(10, 20)); // 30
```

Relationship:

```text
BinaryOperator<T>
       ↓
extends
BiFunction<T,T,T>
```

Common use:

- addition
- max/min
- combining same-type values
- `reduce()` operations

---

# 7. `BiPredicate<T,U>`

Two inputs → boolean.

```java
boolean test(T t, U u);
```

Example:

```java
BiPredicate<String, Integer> hasLength =
        (text, length) -> text.length() == length;

System.out.println(hasLength.test("Java", 4)); // true
```

Think:

```text
T + U → boolean
```

---

# 8. `BiConsumer<T,U>`

Two inputs → no return.

```java
void accept(T t, U u);
```

Example:

```java
BiConsumer<String, Integer> printAge =
        (name, age) ->
                System.out.println(name + " = " + age);

printAge.accept("Nirbhay", 30);
```

Think:

```text
T + U → void
```

---

# 9. `BiFunction<T,U,R>`

Two inputs → one result.

```java
R apply(T t, U u);
```

Example:

```java
BiFunction<Integer, Integer, Integer> add =
        (a, b) -> a + b;

System.out.println(add.apply(10, 20)); // 30
```

Think:

```text
T + U → R
```

---

# 10. Primitive Specializations

Java also provides primitive-specialized interfaces to reduce boxing/unboxing overhead in suitable cases.

Examples:

```text
IntPredicate
IntConsumer
IntSupplier
IntFunction<R>
ToIntFunction<T>
IntUnaryOperator
IntBinaryOperator
```

Example:

```java
IntPredicate positive = number -> number > 0;

System.out.println(positive.test(10));
```

### Interview point

Do not claim that every use of a primitive specialization is automatically faster in every scenario. They are designed to avoid boxing in the interface contract and can be useful in performance-sensitive code.

---

# 11. Composition — `Predicate`

Predicates can be combined.

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

This is heavily used with Streams.

---

# 12. Composition — `Function`

Functions can be chained.

```java
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;

Function<String, String> normalize =
        trim.andThen(upper);

System.out.println(normalize.apply("  java  "));
```

Output:

```text
JAVA
```

### `compose()` vs `andThen()`

```java
f.andThen(g)
```

means:

```text
f → g
```

While:

```java
f.compose(g)
```

means:

```text
g → f
```

This is an important interview question.

---

# 13. `Consumer` Chaining

```java
Consumer<String> print =
        value -> System.out.println(value);

Consumer<String> log =
        value -> System.out.println("LOG: " + value);

Consumer<String> combined = print.andThen(log);

combined.accept("Java");
```

---

# 14. `Supplier` and Lazy Evaluation

A `Supplier` is useful when creation should happen only when requested.

```java
Supplier<String> supplier = () -> {
    System.out.println("Creating value...");
    return "Java";
};

System.out.println("Supplier created");
System.out.println(supplier.get());
```

Creating the `Supplier` does not execute its body. Calling `get()` does.

---

# 15. `Optional` Connection

You will see these interfaces throughout Java 8 APIs.

For example:

```java
Optional<String> value = Optional.of("Java");

value.ifPresent(System.out::println);
```

`ifPresent()` accepts a `Consumer`.

Similarly:

```java
value.map(String::toUpperCase);
```

uses a `Function`.

Optional will be covered deeply later in Chapter 9.

---

# 16. Stream Connection

These interfaces are foundational to Streams.

```java
list.stream()
    .filter(name -> name.length() > 5)
    .map(String::toUpperCase)
    .forEach(System.out::println);
```

Conceptually:

```text
filter → Predicate
map    → Function
forEach→ Consumer
```

This mapping is extremely important for interviews.

---

# 17. Complete Interview Practice Code

## Practice 1 — All Core Interfaces

```java
import java.util.function.*;

public class BuiltInFunctionalInterfacesDemo {

    public static void main(String[] args) {

        Predicate<Integer> isEven =
                number -> number % 2 == 0;

        Consumer<String> print =
                value -> System.out.println("Consumer: " + value);

        Supplier<String> supplier =
                () -> "Supplier Value";

        Function<String, Integer> length =
                String::length;

        System.out.println("Predicate: " + isEven.test(10));
        print.accept("Java");
        System.out.println("Supplier: " + supplier.get());
        System.out.println("Function: " + length.apply("Java"));
    }
}
```

## Practice 2 — Unary & Binary Operators

```java
import java.util.function.*;

public class OperatorDemo {

    public static void main(String[] args) {

        UnaryOperator<Integer> square =
                number -> number * number;

        BinaryOperator<Integer> add =
                (a, b) -> a + b;

        System.out.println("Square: " + square.apply(5));
        System.out.println("Add: " + add.apply(10, 20));
    }
}
```

## Practice 3 — Bi Interfaces

```java
import java.util.function.*;

public class BiFunctionalInterfacesDemo {

    public static void main(String[] args) {

        BiPredicate<String, Integer> validLength =
                (text, length) -> text.length() == length;

        BiConsumer<String, Integer> printAge =
                (name, age) ->
                        System.out.println(name + " = " + age);

        BiFunction<Integer, Integer, Integer> multiply =
                (a, b) -> a * b;

        System.out.println(validLength.test("Java", 4));
        printAge.accept("Nirbhay", 30);
        System.out.println(multiply.apply(10, 5));
    }
}
```

## Practice 4 — Predicate Composition

```java
import java.util.function.Predicate;

public class PredicateCompositionDemo {

    public static void main(String[] args) {

        Predicate<Integer> positive = number -> number > 0;
        Predicate<Integer> even = number -> number % 2 == 0;

        Predicate<Integer> positiveAndEven =
                positive.and(even);

        Predicate<Integer> positiveOrEven =
                positive.or(even);

        Predicate<Integer> notPositive =
                positive.negate();

        System.out.println(positiveAndEven.test(10));
        System.out.println(positiveOrEven.test(-10));
        System.out.println(notPositive.test(-5));
    }
}
```

## Practice 5 — Function Composition

```java
import java.util.function.Function;

public class FunctionCompositionDemo {

    public static void main(String[] args) {

        Function<String, String> trim = String::trim;
        Function<String, String> upper = String::toUpperCase;

        Function<String, String> normalize =
                trim.andThen(upper);

        System.out.println(normalize.apply("  java  "));
    }
}
```

## Practice 6 — Employee Real-World Example ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.function.*;

class Employee {

    private final String name;
    private final double salary;

    Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }
}

public class EmployeeFunctionalInterfaceDemo {

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee("Amit", 50000),
                new Employee("Nirbhay", 90000),
                new Employee("Rahul", 120000)
        );

        Predicate<Employee> highSalary =
                employee -> employee.getSalary() > 80000;

        Function<Employee, String> getName =
                Employee::getName;

        Consumer<String> print =
                System.out::println;

        employees.stream()
                .filter(highSalary)
                .map(getName)
                .forEach(print);
    }
}
```

Output:

```text
Nirbhay
Rahul
```

This single example connects:

```text
Predicate → filter
Function  → map
Consumer  → forEach
```

---

# 18. Quick Comparison Table ⭐⭐⭐⭐⭐

| Interface | Input | Output | Main Method | Typical Use |
|---|---|---|---|---|
| `Predicate<T>` | 1 | boolean | `test()` | validation/filtering |
| `Consumer<T>` | 1 | void | `accept()` | action/logging |
| `Supplier<T>` | 0 | T | `get()` | supply/lazy creation |
| `Function<T,R>` | 1 | R | `apply()` | transformation |
| `UnaryOperator<T>` | 1 | T | `apply()` | same-type transformation |
| `BinaryOperator<T>` | 2 | T | `apply()` | same-type combination |
| `BiPredicate<T,U>` | 2 | boolean | `test()` | two-input condition |
| `BiConsumer<T,U>` | 2 | void | `accept()` | two-input action |
| `BiFunction<T,U,R>` | 2 | R | `apply()` | two-input transformation |

### Memory shortcut

```text
P → test → boolean
C → accept → void
S → get → value
F → apply → transformed value
```

---

# 19. Interview Questions 🔥

### Q1. Difference between `Predicate` and `Function`?

`Predicate` returns `boolean`; `Function` transforms an input into an output.

### Q2. Difference between `Consumer` and `Supplier`?

`Consumer` accepts input and returns nothing; `Supplier` accepts no input and supplies a value.

### Q3. `UnaryOperator` vs `Function`?

`UnaryOperator<T>` is a specialized `Function<T,T>` where input and output have the same type.

### Q4. `BinaryOperator` vs `BiFunction`?

`BinaryOperator<T>` is a specialized `BiFunction<T,T,T>` where both inputs and the result have the same type.

### Q5. What is `BiPredicate`?

A two-input functional interface that returns boolean.

### Q6. What does `Predicate.and()` do?

Creates a predicate that requires both predicates to be true.

### Q7. `Function.andThen()` vs `compose()`?

`andThen`: current function first, then supplied function. `compose`: supplied function first, then current function.

### Q8. Why use primitive functional interfaces?

They provide primitive-specific contracts and can avoid boxing/unboxing in suitable situations.

### Q9. Which functional interface does `Stream.filter()` expect?

`Predicate`.

### Q10. Which does `Stream.map()` expect?

`Function`.

### Q11. Which does `forEach()` expect?

`Consumer`.

### Q12. Which interface would you use for lazy object creation?

Usually `Supplier<T>`.

---

# 20. Common Interview Traps ❌

### Trap 1

`Consumer` returns the consumed value.

❌ Wrong — it returns `void`.

### Trap 2

`Supplier` accepts an input.

❌ Wrong — `Supplier.get()` takes no arguments.

### Trap 3

`Predicate` returns any object.

❌ Wrong — it returns `boolean`.

### Trap 4

`UnaryOperator<T>` can return a different type.

❌ Wrong — its output is the same type `T`.

### Trap 5

`BinaryOperator<T>` means two different input types.

❌ Wrong — both inputs and output are the same `T`.

### Trap 6

`Function.compose()` and `andThen()` execute in the same order.

❌ Wrong — their execution order is reversed.

---

# 21. Coding Challenges 🔥

Try without looking at the solution.

### Challenge 1 — Employee Validation

Use `Predicate<Employee>` to validate:

```text
salary > 50000
age >= 18
```

Combine them.

### Challenge 2 — Employee Transformation

Use `Function<Employee, String>` to convert employees into employee names.

### Challenge 3 — Logging

Use `Consumer<Employee>` to print employee details.

### Challenge 4 — ID Generation

Use `Supplier<String>` to generate a unique request ID.

### Challenge 5 — Price Calculation ⭐⭐⭐⭐⭐

Use:

```text
Function<Double, Double> discount
Function<Double, Double> tax
```

Compose them into a price pipeline.

### Challenge 6 — Two-Input Validation

Use `BiPredicate<String, String>` to check whether two strings are equal ignoring case.

### Challenge 7 — Strategy Combination ⭐⭐⭐⭐⭐

Use `BinaryOperator<Integer>` to create:

```text
max
min
sum
```

strategies.

---

# 22. 60-Second Interview Answer 🧠

> “Java 8 provides built-in functional interfaces in `java.util.function` so we don't need to create custom interfaces for common use cases. `Predicate` takes one input and returns boolean, `Consumer` takes an input and returns nothing, `Supplier` takes no input and produces a value, and `Function` transforms one input into another output. `UnaryOperator` and `BinaryOperator` are specialized Function types where the input and output types remain the same. There are also BiPredicate, BiConsumer and BiFunction for two inputs. These interfaces are heavily used by Streams, Optional and other Java 8 APIs.”

---

# 🧪 Practice Code

urlGitHub — Built-in Functional Interfaces Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/03-Built-in-Functional-Interfaces

---

## Navigation

[← 9.2 — Functional Interfaces](../02-Functional-Interfaces/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.3 — Built-in Functional Interfaces → ✅ Completed**

**Next → 9.4 — Predicate / Consumer / Supplier / Function**