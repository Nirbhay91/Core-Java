# 9.1 — Lambda Expressions Fundamentals

> **Goal:** Understand Lambda Expressions deeply enough to explain them and write common interview code from memory.

## 🎯 Why Lambda Expressions?

Before Java 8, passing behavior to a method commonly required an anonymous class:

```java
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running task");
    }
};
```

Java 8 lets us express the same behavior more concisely:

```java
Runnable task = () -> System.out.println("Running task");
```

### Interview line

> **A lambda expression is a concise way to represent an implementation of a functional interface.**

A lambda does **not** create a standalone function in Java. It provides the implementation of the abstract method of a functional interface.

---

# 1. Lambda Syntax

General form:

```java
(parameters) -> expression
```

or:

```java
(parameters) -> {
    statements;
}
```

Examples:

```java
() -> System.out.println("Hello");

x -> x * 2;

(x, y) -> x + y;

(x, y) -> {
    int result = x + y;
    return result;
};
```

## Syntax rules

### No parameter

```java
() -> System.out.println("Hello");
```

### One parameter

Parentheses are optional:

```java
x -> x * x
```

Equivalent:

```java
(x) -> x * x
```

### Multiple parameters

Parentheses are required:

```java
(a, b) -> a + b
```

### Explicit parameter types

```java
(int a, int b) -> a + b
```

Do not mix styles:

```java
// Invalid
(int a, b) -> a + b
```

---

# 2. Lambda Needs a Target Type

This is one of the most important interview concepts.

A lambda requires a **target type**, normally a functional interface.

Valid:

```java
Runnable r = () -> System.out.println("Hello");
```

Here the target type is `Runnable`.

Invalid as a standalone statement:

```java
() -> System.out.println("Hello");
```

### Interview line

> **A lambda expression gets its type from the target functional interface; it is not itself a standalone object type.**

---

# 3. Functional Interface Requirement

A lambda can implement a functional interface — an interface with exactly **one abstract method**.

Example:

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
}
```

Use lambda:

```java
Calculator add = (a, b) -> a + b;

System.out.println(add.calculate(10, 20));
```

Output:

```text
30
```

### Important

A functional interface may still contain:

- one abstract method
- any number of `default` methods
- any number of `static` methods
- methods inherited from `Object` do not count as additional abstract methods

---

# 4. Lambda vs Anonymous Class

### Anonymous class

```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};
```

### Lambda

```java
Runnable r = () -> System.out.println("Hello");
```

Lambda is shorter and focuses on **behavior rather than boilerplate**.

---

# 5. Lambda with `Comparator`

Before Java 8:

```java
Comparator<String> comparator = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};
```

Lambda:

```java
Comparator<String> comparator =
        (a, b) -> a.length() - b.length();
```

Interview-friendly version:

```java
Comparator<String> comparator =
        Comparator.comparingInt(String::length);
```

The method-reference version will be covered in **9.5**.

---

# 6. Lambda with Collection

```java
List<String> names = Arrays.asList("Nirbhay", "Amit", "Rahul");

names.forEach(name -> System.out.println(name));
```

This is a common Java 8 interview example.

---

# 7. Lambda Capturing Local Variables

A lambda can access local variables only when they are **final or effectively final**.

Valid:

```java
int multiplier = 10;

Function<Integer, Integer> function =
        number -> number * multiplier;
```

Invalid:

```java
int multiplier = 10;

Function<Integer, Integer> function =
        number -> number * multiplier;

multiplier = 20; // compile-time error because multiplier is no longer effectively final
```

### Interview line

> **A lambda can capture a local variable only when that variable is final or effectively final.**

Do not confuse this with instance fields; instance fields can be modified because they belong to the object rather than being captured local variables.

---

# 8. Lambda and `this`

This is a popular interview question.

Inside a lambda, `this` refers to the **enclosing object**, unlike an anonymous inner class where `this` refers to the anonymous class instance.

Example:

```java
class Demo {

    private String name = "Nirbhay";

    void test() {
        Runnable lambda = () ->
                System.out.println(this.name);

        lambda.run();
    }
}
```

Output:

```text
Nirbhay
```

### Interview line

> **A lambda does not introduce a new `this`; it uses the `this` of the enclosing context.**

---

# 9. Return Value

Expression lambda:

```java
Function<Integer, Integer> square = x -> x * x;
```

Block lambda:

```java
Function<Integer, Integer> square = x -> {
    int result = x * x;
    return result;
};
```

If a block lambda has a return type, the appropriate `return` statement is required.

---

# 10. Real-World Example — Employee Filtering

```java
List<Employee> employees = Arrays.asList(
        new Employee(1, "Amit", 50000),
        new Employee(2, "Nirbhay", 80000),
        new Employee(3, "Rahul", 45000)
);

employees.forEach(employee -> {
    if (employee.getSalary() > 60000) {
        System.out.println(employee.getName());
    }
});
```

This demonstrates passing behavior into `forEach()`.

A more idiomatic Stream-based solution will be covered in Chapter 9's Stream topics.

---

# 11. Lambda Does Not Mean Multithreading

This is a common misconception.

```java
Runnable task = () -> System.out.println("Hello");
```

Creating a lambda does **not** automatically create a new thread.

A thread is created only when the `Runnable` is executed through a thread mechanism, for example:

```java
new Thread(task).start();
```

### Interview trap

**Q: Does a lambda create a new thread?**

**Answer:** No. Lambda represents behavior. The execution mechanism decides whether it runs on the current thread, a new thread, or an executor thread.

---

# 12. Lambda vs Method Reference

Lambda:

```java
name -> System.out.println(name)
```

Method reference:

```java
System.out::println
```

Both can represent the same behavior in a compatible functional-interface context.

Method references will be covered in **9.5**.

---

# 13. Complete Interview Practice Code

## Practice 1 — Basic Lambda

```java
public class LambdaBasicDemo {

    public static void main(String[] args) {

        Runnable task = () ->
                System.out.println("Running Lambda");

        task.run();
    }
}
```

## Practice 2 — Custom Functional Interface

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
}

public class LambdaCalculatorDemo {

    public static void main(String[] args) {

        Calculator add = (a, b) -> a + b;
        Calculator multiply = (a, b) -> a * b;

        System.out.println("Addition: " + add.calculate(10, 20));
        System.out.println("Multiplication: " + multiply.calculate(10, 20));
    }
}
```

## Practice 3 — Lambda with Comparator

```java
import java.util.*;

public class LambdaComparatorDemo {

    public static void main(String[] args) {

        List<String> names = new ArrayList<>(
                Arrays.asList("Nirbhay", "Amit", "Rahul", "Raj")
        );

        names.sort((a, b) -> a.length() - b.length());

        System.out.println(names);
    }
}
```

## Practice 4 — Lambda with Thread

```java
public class LambdaThreadDemo {

    public static void main(String[] args) {

        Runnable task = () -> {
            System.out.println("Task running on: "
                    + Thread.currentThread().getName());
        };

        Thread thread = new Thread(task);
        thread.start();
    }
}
```

## Practice 5 — Lambda Capturing Effectively Final Variable

```java
import java.util.function.Function;

public class LambdaCaptureDemo {

    public static void main(String[] args) {

        int multiplier = 10;

        Function<Integer, Integer> multiply =
                number -> number * multiplier;

        System.out.println(multiply.apply(5));
    }
}
```

---

# 14. Interview Questions

### Q1. What is a lambda expression?

**Answer:** A concise expression used to provide an implementation of a functional interface's single abstract method.

### Q2. Why was lambda introduced?

**Answer:** To reduce boilerplate and make behavior easier to pass as data, enabling functional-style programming and APIs such as Streams.

### Q3. Can lambda exist without a functional interface?

**Answer:** A lambda needs a target type; in normal Java usage that target is a functional interface.

### Q4. Can a functional interface have default methods?

**Answer:** Yes. It must still have exactly one abstract method.

### Q5. Can a lambda modify a local variable?

**Answer:** It cannot capture and modify a local variable because captured locals must be final or effectively final.

### Q6. What does `this` mean inside a lambda?

**Answer:** `this` refers to the enclosing object; lambda does not create a new `this` scope.

### Q7. Does lambda create a new object?

**Answer:** Do not describe it as simply “lambda creates an object.” At the language level it is an expression whose target type is a functional interface; runtime implementation details can vary.

### Q8. Does lambda create a thread?

**Answer:** No. It represents behavior; a thread/executor determines how the behavior is executed.

### Q9. Lambda vs anonymous inner class?

**Answer:** Lambda is designed for functional-interface behavior and has different `this`/scope semantics; anonymous classes can define more general class bodies and their own instance state.

### Q10. Can lambda throw checked exceptions?

**Answer:** Only when the functional interface's abstract method declares compatible checked exceptions. A lambda cannot throw a checked exception that the target method contract does not allow.

---

# 15. Common Mistakes ❌

### Mistake 1

Thinking lambda is a replacement for every anonymous class.

**Correct:** Lambda targets functional interfaces; anonymous classes can represent more general class/interface implementations.

### Mistake 2

Thinking lambda automatically runs asynchronously.

**Correct:** Lambda only represents behavior.

### Mistake 3

Changing a captured local variable:

```java
int x = 10;
Runnable r = () -> System.out.println(x);
x = 20; // ❌
```

### Mistake 4

Thinking `this` inside lambda refers to a new lambda object.

**Correct:** It refers to the enclosing instance.

---

# 16. Coding Challenges 🔥

Try these **without looking at the examples above**.

### Challenge 1
Create a functional interface:

```java
interface MathOperation {
    int operate(int a, int b);
}
```

Implement:

```text
addition
subtraction
multiplication
division
```

using lambdas.

### Challenge 2
Create a `Runnable` using lambda that prints the current thread name.

### Challenge 3
Sort a list of employees by salary using a lambda comparator.

### Challenge 4
Create a lambda that checks whether a number is even.

### Challenge 5
Create a lambda that converts a string to uppercase.

### Challenge 6 — Interview Level ⭐⭐⭐⭐⭐
Create a custom functional interface:

```java
interface StringProcessor {
    String process(String value);
}
```

Implement using lambda:

```text
trim
uppercase
reverse
```

Then explain why these implementations are behavior passed as data.

---

# 17. 60-Second Interview Answer 🧠

> “Lambda expressions were introduced in Java 8 to reduce boilerplate and enable functional-style programming. A lambda provides an implementation of a functional interface's single abstract method. The syntax is `(parameters) -> expression` or `(parameters) -> { statements }`. A lambda needs a target type, so it is commonly assigned to a functional interface such as `Runnable`, `Comparator`, `Predicate`, or a custom interface. Captured local variables must be final or effectively final, and lambda does not create a new `this`; `this` refers to the enclosing object. Also, lambda itself does not create a thread—it only represents behavior.”

---

# 🧪 Practice Code

**Complete runnable practice:**

urlGitHub — Lambda Expressions Fundamentals Practicehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/01-Lambda-Expressions-Fundamentals

---

## Navigation

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.1 — Lambda Expressions Fundamentals → ✅ Completed**

**Next → 9.2 — Functional Interfaces**