# 9.2 — Functional Interfaces

> **Goal:** Understand functional interfaces deeply enough to design your own, use Java's built-in interfaces, explain `@FunctionalInterface`, and write interview code from memory.

## 🎯 Interview Definition

> **A functional interface is an interface that has exactly one abstract method, making it eligible to be used as the target type of a lambda expression or method reference.**

Example:

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
}
```

Lambda:

```java
Calculator add = (a, b) -> a + b;
```

---

# 1. What Makes an Interface Functional?

The key rule is:

```text
Exactly ONE abstract method
```

It can still have:

- multiple `default` methods
- multiple `static` methods
- methods that override `Object` methods such as `equals()` / `toString()` do not count as additional abstract methods

Example:

```java
@FunctionalInterface
interface EmployeeProcessor {

    void process(String name);       // only abstract method

    default void log() {
        System.out.println("Processing...");
    }

    static void info() {
        System.out.println("Employee processor");
    }
}
```

This is still a functional interface.

---

# 2. `@FunctionalInterface`

`@FunctionalInterface` is an annotation that tells the compiler and developers that the interface is intended to have exactly one abstract method.

```java
@FunctionalInterface
interface Calculator {
    int add(int a, int b);
}
```

If someone later adds another abstract method:

```java
@FunctionalInterface
interface Calculator {
    int add(int a, int b);
    int subtract(int a, int b); // compile-time error
}
```

### Important interview point

> `@FunctionalInterface` is **not mandatory** for an interface to be functional. The compiler can still treat an interface as functional based on its method structure.

The annotation provides compile-time validation and communicates intent.

---

# 3. Why Functional Interfaces Were Introduced

Java 8 needed a clean way to pass behavior to APIs.

Before Java 8:

```java
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running");
    }
};
```

With Java 8:

```java
Runnable task = () -> System.out.println("Running");
```

The functional interface provides the **target type** for the lambda.

---

# 4. Custom Functional Interface

```java
@FunctionalInterface
interface MathOperation {
    int operate(int a, int b);
}
```

Use different behaviors:

```java
MathOperation add = (a, b) -> a + b;
MathOperation subtract = (a, b) -> a - b;
MathOperation multiply = (a, b) -> a * b;
```

Execute:

```java
System.out.println(add.operate(10, 5));
System.out.println(subtract.operate(10, 5));
System.out.println(multiply.operate(10, 5));
```

Output:

```text
15
5
50
```

This is the core idea of **passing behavior as data**.

---

# 5. Functional Interface Can Have Default Methods

```java
@FunctionalInterface
interface PaymentProcessor {

    void process(double amount);

    default void audit() {
        System.out.println("Audit completed");
    }
}
```

Use:

```java
PaymentProcessor processor =
        amount -> System.out.println("Paid: " + amount);

processor.process(1000);
processor.audit();
```

`audit()` does not make the interface non-functional because it is a `default` method, not an abstract method.

---

# 6. Functional Interface Can Have Static Methods

```java
@FunctionalInterface
interface Validator {

    boolean validate(String value);

    static boolean isNull(String value) {
        return value == null;
    }
}
```

Call static method through the interface:

```java
Validator.isNull("Nirbhay");
```

The static method does not count as the abstract method.

---

# 7. `Object` Methods and Functional Interface Rule

This is a common interview trap.

```java
@FunctionalInterface
interface EmployeeAction {

    void execute();

    @Override
    boolean equals(Object obj);
}
```

This remains functional because `equals(Object)` corresponds to a method from `Object` and does not introduce another abstract contract for functional-interface purposes.

### Interview line

> **The functional-interface test is based on the interface's abstract method contract after accounting for inherited `Object` methods.**

---

# 8. Built-in Functional Interfaces

Java 8 provides many ready-made functional interfaces in `java.util.function`.

| Interface | Main Method | Input | Output |
|---|---|---|---|
| `Predicate<T>` | `test(T)` | T | boolean |
| `Consumer<T>` | `accept(T)` | T | void |
| `Supplier<T>` | `get()` | none | T |
| `Function<T,R>` | `apply(T)` | T | R |
| `UnaryOperator<T>` | `apply(T)` | T | T |
| `BinaryOperator<T>` | `apply(T,T)` | T,T | T |
| `BiPredicate<T,U>` | `test(T,U)` | T,U | boolean |
| `BiConsumer<T,U>` | `accept(T,U)` | T,U | void |
| `BiFunction<T,U,R>` | `apply(T,U)` | T,U | R |

These will be explored in depth in **9.3 and 9.4**.

---

# 9. Predicate Example

```java
Predicate<Integer> even = number -> number % 2 == 0;

System.out.println(even.test(10)); // true
System.out.println(even.test(7));  // false
```

Think:

```text
Input → condition → boolean
```

---

# 10. Consumer Example

```java
Consumer<String> printer =
        name -> System.out.println("Hello " + name);

printer.accept("Nirbhay");
```

Think:

```text
Input → action → no return
```

---

# 11. Supplier Example

```java
Supplier<Double> randomValue =
        () -> Math.random();

System.out.println(randomValue.get());
```

Think:

```text
No input → produces value
```

---

# 12. Function Example

```java
Function<String, Integer> length =
        value -> value.length();

System.out.println(length.apply("Java"));
```

Think:

```text
T → R
```

---

# 13. Functional Interface Composition

Functional interfaces can be combined.

Example with `Predicate`:

```java
Predicate<Integer> positive = number -> number > 0;
Predicate<Integer> even = number -> number % 2 == 0;

Predicate<Integer> positiveAndEven =
        positive.and(even);

System.out.println(positiveAndEven.test(10)); // true
System.out.println(positiveAndEven.test(-10)); // false
```

This becomes important when working with Streams.

---

# 14. Passing Functional Interface to a Method

This is an important design pattern.

```java
@FunctionalInterface
interface Operation {
    int execute(int a, int b);
}

public class CalculatorService {

    static int calculate(int a, int b, Operation operation) {
        return operation.execute(a, b);
    }

    public static void main(String[] args) {

        int result = calculate(10, 20, (x, y) -> x + y);

        System.out.println(result);
    }
}
```

Output:

```text
30
```

### Interview line

> **A functional interface lets a method receive behavior as an argument.**

---

# 15. Returning a Functional Interface

A method can also return behavior.

```java
@FunctionalInterface
interface Operation {
    int apply(int a, int b);
}

public class OperationFactory {

    static Operation getOperation(String type) {

        if ("ADD".equalsIgnoreCase(type)) {
            return (a, b) -> a + b;
        }

        if ("MULTIPLY".equalsIgnoreCase(type)) {
            return (a, b) -> a * b;
        }

        throw new IllegalArgumentException("Unknown operation");
    }

    public static void main(String[] args) {

        Operation operation = getOperation("ADD");

        System.out.println(operation.apply(10, 20));
    }
}
```

This pattern is useful for strategy-style designs.

---

# 16. Functional Interface vs Normal Interface

| Feature | Normal Interface | Functional Interface |
|---|---|---|
| Abstract methods | One or more | Exactly one |
| Lambda target | ❌ Not generally | ✅ Yes |
| `default` methods | ✅ | ✅ |
| `static` methods | ✅ | ✅ |
| `@FunctionalInterface` | Not required | Recommended |
| Main purpose | Contract | Contract + behavior as lambda |

### Important correction

A functional interface is still an interface. It is not a different Java type.

The important difference is its **single abstract method** contract.

---

# 17. Functional Interface vs Abstract Class

| Functional Interface | Abstract Class |
|---|---|
| Can have one abstract method | Can have multiple abstract methods |
| Lambda-compatible | Not lambda-compatible |
| Multiple interfaces can be implemented | Only one class can be extended |
| No instance fields in the interface contract | Can have instance state |
| Best for behavior/strategy contracts | Best for shared state + behavior |

---

# 18. Complete Interview Practice Code

## Practice 1 — Custom Functional Interface

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
}

public class FunctionalInterfaceBasicDemo {

    public static void main(String[] args) {

        Calculator add = (a, b) -> a + b;
        Calculator multiply = (a, b) -> a * b;

        System.out.println("Addition: " + add.calculate(10, 20));
        System.out.println("Multiplication: " + multiply.calculate(10, 20));
    }
}
```

## Practice 2 — Default + Static Methods

```java
@FunctionalInterface
interface MessageProcessor {

    void process(String message);

    default void log() {
        System.out.println("Processing message");
    }

    static void info() {
        System.out.println("MessageProcessor interface");
    }
}

public class FunctionalInterfaceMethodsDemo {

    public static void main(String[] args) {

        MessageProcessor processor =
                message -> System.out.println("Message: " + message);

        processor.process("Hello Java 8");
        processor.log();
        MessageProcessor.info();
    }
}
```

## Practice 3 — Pass Behavior to a Method

```java
@FunctionalInterface
interface Operation {
    int apply(int a, int b);
}

public class FunctionalInterfaceAsParameterDemo {

    static int calculate(int a, int b, Operation operation) {
        return operation.apply(a, b);
    }

    public static void main(String[] args) {

        System.out.println(
                calculate(10, 5, (a, b) -> a + b)
        );

        System.out.println(
                calculate(10, 5, (a, b) -> a * b)
        );
    }
}
```

## Practice 4 — Return Behavior from Method

```java
@FunctionalInterface
interface Operation {
    int apply(int a, int b);
}

public class FunctionalInterfaceReturnDemo {

    static Operation getOperation(String type) {

        switch (type.toUpperCase()) {
            case "ADD":
                return (a, b) -> a + b;
            case "SUBTRACT":
                return (a, b) -> a - b;
            case "MULTIPLY":
                return (a, b) -> a * b;
            default:
                throw new IllegalArgumentException("Invalid operation");
        }
    }

    public static void main(String[] args) {

        Operation operation = getOperation("MULTIPLY");

        System.out.println(operation.apply(10, 20));
    }
}
```

## Practice 5 — Predicate Composition

```java
import java.util.function.Predicate;

public class FunctionalInterfacePredicateDemo {

    public static void main(String[] args) {

        Predicate<Integer> positive = number -> number > 0;
        Predicate<Integer> even = number -> number % 2 == 0;

        Predicate<Integer> positiveAndEven =
                positive.and(even);

        System.out.println(positiveAndEven.test(10));
        System.out.println(positiveAndEven.test(-10));
        System.out.println(positiveAndEven.test(7));
    }
}
```

---

# 19. Interview Questions 🔥

### Q1. What is a functional interface?

An interface with exactly one abstract method, suitable as a lambda/method-reference target.

### Q2. Is `@FunctionalInterface` mandatory?

No. It is optional but recommended because it makes the compiler enforce the intended functional-interface contract.

### Q3. Can a functional interface contain default methods?

Yes. Any number of default methods are allowed.

### Q4. Can a functional interface contain static methods?

Yes. Any number of static methods are allowed.

### Q5. Can it override `Object` methods?

Yes. Methods corresponding to `Object` methods do not make the interface non-functional.

### Q6. Why does Java need functional interfaces for lambdas?

Because the lambda needs a target type and the functional interface supplies the single abstract-method contract that the lambda implements.

### Q7. Can an abstract class be a lambda target?

No. Lambdas target functional interfaces, not abstract classes.

### Q8. Can an interface have two default methods and still be functional?

Yes, as long as it has exactly one abstract method.

### Q9. What happens if `@FunctionalInterface` has two abstract methods?

Compilation fails.

### Q10. Why are functional interfaces important in Java 8?

They enable behavior to be passed as values and form the foundation of Lambdas, Streams and many Java 8 APIs.

### Q11. Can a functional interface extend another interface?

Yes, provided the resulting interface still has exactly one abstract method.

### Q12. Can a functional interface be generic?

Yes.

Example:

```java
@FunctionalInterface
interface Converter<T, R> {
    R convert(T value);
}
```

---

# 20. Common Interview Traps ❌

### Trap 1 — `@FunctionalInterface` means one method total

❌ Wrong.

It means one **abstract** method. Default and static methods are allowed.

### Trap 2 — Lambda can implement an abstract class

❌ Wrong.

Lambda requires a functional-interface target type.

### Trap 3 — `@FunctionalInterface` is compulsory

❌ Wrong.

The annotation is optional.

### Trap 4 — `default` method counts as abstract method

❌ Wrong.

It already has an implementation.

### Trap 5 — Static method can be overridden

❌ Wrong.

Static interface methods belong to the interface and are called through the interface name.

---

# 21. Coding Challenges 🔥

Try these from memory.

### Challenge 1 — Math Strategy

Create:

```java
@FunctionalInterface
interface MathOperation {
    int operate(int a, int b);
}
```

Implement add, subtract, multiply and divide using lambdas.

### Challenge 2 — String Strategy

Create:

```java
@FunctionalInterface
interface StringProcessor {
    String process(String value);
}
```

Implement:

```text
uppercase
lowercase
reverse
trim
```

### Challenge 3 — Generic Converter ⭐⭐⭐⭐⭐

Create:

```java
@FunctionalInterface
interface Converter<T, R> {
    R convert(T value);
}
```

Use it to convert:

```text
String → Integer
Integer → String
String → String length
```

### Challenge 4 — Strategy Pattern ⭐⭐⭐⭐⭐

Create a payment processor that accepts a functional interface as a parameter:

```text
CREDIT_CARD
UPI
NET_BANKING
```

Do not create separate subclasses just for the strategy behavior.

### Challenge 5 — Predicate Composition

Create predicates for:

```text
age >= 18
salary > 50000
city = Bangalore
```

Combine them using `and()` / `or()`.

---

# 22. 60-Second Interview Answer 🧠

> “A functional interface is an interface with exactly one abstract method. Java 8 uses functional interfaces as target types for lambda expressions and method references. `@FunctionalInterface` is optional but helps the compiler validate the contract. A functional interface can still contain multiple default and static methods. Common examples are `Runnable`, `Comparator`, `Predicate`, `Consumer`, `Supplier` and `Function`. The main purpose is to allow behavior to be passed as an argument, which is heavily used by Streams and other Java 8 APIs.”

---

# 🧪 Practice Code

urlGitHub — Functional Interfaces Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/02-Functional-Interfaces

---

## Navigation

[← 9.1 — Lambda Expressions Fundamentals](../01-Lambda-Expressions-Fundamentals/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.2 — Functional Interfaces → ✅ Completed**

**Next → 9.3 — Built-in Functional Interfaces**