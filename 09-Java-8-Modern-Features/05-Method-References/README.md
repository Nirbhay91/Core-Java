# 9.5 — Method References

> **Goal:** Understand Java 8 method references, all four forms, how they map to functional interfaces, when to use them instead of lambdas, and the interview traps around overloaded methods, constructors, and static vs instance methods.

## 🎯 Interview Definition

> A method reference is a shorthand for a lambda expression when the lambda only delegates to an existing method or constructor.

Example:

```java
// Lambda
names.forEach(name -> System.out.println(name));

// Method reference
names.forEach(System.out::println);
```

The method reference does **not execute the method immediately**. It provides a method reference that is invoked when the target functional interface calls it.

---

# 1. Why Method References?

They improve readability when the lambda contains nothing except a direct method call.

```java
Function<String, Integer> length1 =
        text -> text.length();

Function<String, Integer> length2 =
        String::length;
```

Both represent the same functional behavior.

### Interview rule

> If a lambda simply passes its parameter to an existing method, check whether a method reference can replace it.

---

# 2. The Four Types of Method References ⭐⭐⭐⭐⭐

Java supports four main forms:

```text
1. Static method
2. Particular object's instance method
3. Instance method of an arbitrary object of a particular type
4. Constructor reference
```

Memory shortcut:

```text
Class::staticMethod
object::instanceMethod
Class::instanceMethod
Class::new
```

---

# 3. Type 1 — Static Method Reference

Syntax:

```java
ClassName::staticMethodName
```

Example:

```java
Function<Integer, Integer> absolute =
        Math::abs;

System.out.println(absolute.apply(-10)); // 10
```

Equivalent lambda:

```java
Function<Integer, Integer> absolute =
        number -> Math.abs(number);
```

Another example:

```java
BiFunction<Integer, Integer, Integer> max =
        Math::max;
```

Equivalent:

```java
(a, b) -> Math.max(a, b)
```

---

# 4. Type 2 — Instance Method of a Particular Object

Syntax:

```java
object::instanceMethod
```

Example:

```java
String prefix = "Hello ";

Function<String, String> greeting =
        prefix::concat;

System.out.println(greeting.apply("Nirbhay"));
```

Equivalent lambda:

```java
text -> prefix.concat(text)
```

Another example:

```java
Consumer<String> printer =
        System.out::println;
```

Here `System.out` is the particular object and `println` is its instance method.

---

# 5. Type 3 — Instance Method of an Arbitrary Object

Syntax:

```java
ClassName::instanceMethod
```

This can be confusing because it looks similar to a static method reference.

Example:

```java
Function<String, String> upper =
        String::toUpperCase;
```

Equivalent lambda:

```java
text -> text.toUpperCase()
```

The object on which `toUpperCase()` runs is supplied as the functional interface argument.

Another example:

```java
Function<String, Integer> length =
        String::length;
```

Equivalent:

```java
text -> text.length()
```

### Interview distinction

```text
Math::abs
```

→ static method reference.

```text
String::length
```

→ instance method of an arbitrary `String` object.

---

# 6. Type 4 — Constructor Reference

Syntax:

```java
ClassName::new
```

Example:

```java
Supplier<ArrayList<String>> listFactory =
        ArrayList::new;

ArrayList<String> list = listFactory.get();
```

Equivalent lambda:

```java
Supplier<ArrayList<String>> listFactory =
        () -> new ArrayList<>();
```

### Constructor with arguments

```java
Function<String, StringBuilder> builderFactory =
        StringBuilder::new;

StringBuilder builder =
        builderFactory.apply("Java");
```

Equivalent:

```java
text -> new StringBuilder(text)
```

---

# 7. Method References and Functional Interfaces

A method reference needs a **target type**.

For example:

```java
Function<String, Integer> length = String::length;
```

The compiler knows that `String::length` must match:

```java
Integer apply(String value)
```

because `Function<String, Integer>` provides the target functional-interface type.

### Important

This may not compile without enough target-type information:

```java
var ref = String::length; // ❌ invalid
```

A method reference is not itself an object with a standalone type; it needs a target functional-interface type in contexts such as assignment, method invocation, or casting.

---

# 8. Method Reference vs Lambda

### Example 1

```java
// Lambda
name -> System.out.println(name)

// Method reference
System.out::println
```

### Example 2

```java
// Lambda
text -> text.toUpperCase()

// Method reference
String::toUpperCase
```

### Example 3

```java
// Lambda
number -> Math.abs(number)

// Method reference
Math::abs
```

### Rule

If the lambda body is a direct method invocation with no extra logic, a method reference is often a cleaner form.

If there is additional logic, keep the lambda.

```java
// Method reference works
names.forEach(System.out::println);

// Lambda is clearer because there is extra logic
names.forEach(name ->
        System.out.println("User: " + name));
```

---

# 9. Static Method Reference — Complete Example

```java
import java.util.function.BiFunction;
import java.util.function.Function;

public class StaticMethodReferenceDemo {

    public static void main(String[] args) {

        Function<Integer, Integer> absoluteValue =
                Math::abs;

        BiFunction<Integer, Integer, Integer> maximum =
                Math::max;

        System.out.println(absoluteValue.apply(-50));
        System.out.println(maximum.apply(10, 20));
    }
}
```

---

# 10. Particular Object Method Reference

```java
import java.util.function.Consumer;

public class ParticularObjectMethodReferenceDemo {

    public static void main(String[] args) {

        String prefix = "Hello ";

        Consumer<String> printer =
                System.out::println;

        printer.accept("Java");

        // Particular object reference
        java.util.function.Function<String, String> greeting =
                prefix::concat;

        System.out.println(greeting.apply("Nirbhay"));
    }
}
```

---

# 11. Arbitrary Object Instance Method Reference

```java
import java.util.function.Function;

public class ArbitraryObjectMethodReferenceDemo {

    public static void main(String[] args) {

        Function<String, String> upper =
                String::toUpperCase;

        Function<String, Integer> length =
                String::length;

        System.out.println(upper.apply("java"));
        System.out.println(length.apply("Java"));
    }
}
```

---

# 12. Constructor Reference

```java
import java.util.ArrayList;
import java.util.function.Function;
import java.util.function.Supplier;

public class ConstructorReferenceDemo {

    public static void main(String[] args) {

        Supplier<ArrayList<String>> listFactory =
                ArrayList::new;

        ArrayList<String> list = listFactory.get();
        list.add("Java");

        Function<String, StringBuilder> builderFactory =
                StringBuilder::new;

        StringBuilder builder =
                builderFactory.apply("Java");

        System.out.println(list);
        System.out.println(builder);
    }
}
```

---

# 13. Employee Example ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.function.Function;

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

    @Override
    public String toString() {
        return id + " - " + name + " - " + salary;
    }
}

public class EmployeeMethodReferenceDemo {

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Amit", 50000),
                new Employee(2, "Nirbhay", 90000),
                new Employee(3, "Rahul", 120000)
        );

        // Constructor reference
        Function<String, Employee> employeeFactory =
                name -> new Employee(100, name, 50000);

        // Instance method reference
        employees.stream()
                .map(Employee::getName)
                .forEach(System.out::println);

        // Static method reference can be used similarly
        employees.stream()
                .map(Employee::getSalary)
                .forEach(System.out::println);
    }
}
```

The important references here are:

```text
Employee::getName
Employee::getSalary
System.out::println
```

---

# 14. Method References with Streams ⭐⭐⭐⭐⭐

Very common interview code:

```java
List<String> names = Arrays.asList(
        "Nirbhay", "Amit", "Rahul"
);

names.stream()
        .map(String::toUpperCase)
        .forEach(System.out::println);
```

Equivalent lambda version:

```java
names.stream()
        .map(name -> name.toUpperCase())
        .forEach(name -> System.out.println(name));
```

### Interview explanation

```text
map(String::toUpperCase)
        ↓
Function<String, String>

forEach(System.out::println)
        ↓
Consumer<String>
```

---

# 15. Method References with Sorting

```java
List<String> names = new ArrayList<>(
        Arrays.asList("Rahul", "Amit", "Nirbhay")
);

names.sort(String::compareTo);

System.out.println(names);
```

Equivalent:

```java
names.sort((a, b) -> a.compareTo(b));
```

For reverse order:

```java
names.sort(Comparator.reverseOrder());
```

---

# 16. Static Method Reference with `Comparator`

```java
List<Integer> numbers =
        new ArrayList<>(Arrays.asList(10, 5, 20, 1));

numbers.sort(Integer::compare);

System.out.println(numbers);
```

Equivalent:

```java
numbers.sort((a, b) -> Integer.compare(a, b));
```

---

# 17. Constructor Reference with Custom Class

```java
import java.util.function.Function;

class User {

    private final String name;

    User(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "User{name='" + name + "'}";
    }
}

public class UserFactoryDemo {

    public static void main(String[] args) {

        Function<String, User> factory =
                User::new;

        User user = factory.apply("Nirbhay");

        System.out.println(user);
    }
}
```

Think:

```text
Function<String, User>
        ↓
String input
        ↓
User::new
        ↓
new User(String)
```

---

# 18. Overloaded Methods — Interview Trap 🔥

Method references can become ambiguous when overloaded methods exist.

Example:

```java
class Printer {

    void print(String value) {
        System.out.println("String: " + value);
    }

    void print(Integer value) {
        System.out.println("Integer: " + value);
    }
}
```

The target functional-interface type can provide the parameter type needed to resolve the method reference.

For example:

```java
Printer printer = new Printer();

Consumer<String> stringPrinter = printer::print;
Consumer<Integer> integerPrinter = printer::print;
```

The functional interface's method signature gives the compiler the necessary target type.

---

# 19. Bound vs Unbound Instance Method Reference

This is a common advanced interview topic.

### Bound

```java
String text = "Java";

Supplier<Integer> length =
        text::length;
```

The receiver object is already fixed:

```text
text → length()
```

### Unbound / arbitrary receiver

```java
Function<String, Integer> length =
        String::length;
```

The receiver object is supplied as the function argument:

```text
String argument → length()
```

### Memory

```text
object::method  → bound
Class::method   → receiver supplied by functional argument
```

---

# 20. `this::method` and `super::method`

Method references can also refer to methods of the current object and superclass.

```java
class Service {

    void process(String value) {
        System.out.println(value);
    }

    void execute() {
        java.util.function.Consumer<String> consumer =
                this::process;

        consumer.accept("Java");
    }
}
```

And superclass method reference can use:

```java
super::methodName
```

These are useful when passing existing object behavior as a functional value.

---

# 21. Method Reference Does Not Mean Immediate Execution

Important:

```java
Consumer<String> printer = System.out::println;
```

At this line, `println()` is not being called with a value.

It is being used to create the behavior represented by the `Consumer`.

Actual invocation happens here:

```java
printer.accept("Java");
```

Output:

```text
Java
```

---

# 22. Complete Interview Practice — All Four Types ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.function.*;

class Employee {

    private final String name;

    Employee(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    @Override
    public String toString() {
        return "Employee{" + name + "}";
    }
}

public class AllMethodReferencesDemo {

    public static void main(String[] args) {

        // 1. Static method reference
        Function<Integer, Integer> absolute = Math::abs;
        System.out.println(absolute.apply(-100));

        // 2. Particular object's instance method
        String prefix = "Hello ";
        Function<String, String> greeting = prefix::concat;
        System.out.println(greeting.apply("Nirbhay"));

        // 3. Arbitrary object's instance method
        Function<String, String> upper = String::toUpperCase;
        System.out.println(upper.apply("java"));

        // 4. Constructor reference
        Function<String, Employee> employeeFactory = Employee::new;
        Employee employee = employeeFactory.apply("Nirbhay");
        System.out.println(employee);

        // Stream example
        List<String> names =
                Arrays.asList("Nirbhay", "Amit", "Rahul");

        names.stream()
                .map(String::toUpperCase)
                .forEach(System.out::println);
    }
}
```

---

# 23. Method Reference Cheat Sheet 🧠

| Type | Syntax | Example | Equivalent Lambda |
|---|---|---|---|
| Static | `Class::staticMethod` | `Math::abs` | `x -> Math.abs(x)` |
| Particular object | `object::method` | `System.out::println` | `x -> System.out.println(x)` |
| Arbitrary object | `Class::method` | `String::toUpperCase` | `x -> x.toUpperCase()` |
| Constructor | `Class::new` | `User::new` | `x -> new User(x)` |

### ⭐ One-line memory

```text
Class::staticMethod  → static
object::method       → particular object
Class::method        → instance method, receiver comes from input
Class::new           → constructor
```

---

# 24. Interview Questions 🔥

### Q1. What is a method reference?

A shorthand for a lambda that directly delegates to an existing method or constructor.

### Q2. How many types of method references are there?

Four main forms: static method, particular-object instance method, arbitrary-object instance method, and constructor reference.

### Q3. Is a method reference executed immediately?

No. It represents behavior and is invoked through the target functional interface.

### Q4. Why does a method reference need a target type?

The compiler uses the target functional-interface signature to determine which method/constructor signature the reference must represent.

### Q5. `String::length` — static or instance?

Instance method of an arbitrary `String` object.

### Q6. `System.out::println` — which type?

Instance method of a particular object (`System.out`).

### Q7. `User::new`?

Constructor reference.

### Q8. `Math::abs`?

Static method reference.

### Q9. Can overloaded methods be used with method references?

Yes, when the target type provides enough information to resolve the overload; otherwise the reference can be ambiguous.

### Q10. What is the difference between `object::method` and `Class::method` for instance methods?

`object::method` binds the receiver object in advance; `Class::method` represents an instance method where the receiver is supplied by the functional-interface argument.

### Q11. Can method references be used with Streams?

Yes, extensively with operations such as `map`, `forEach`, `sorted`, and collectors.

### Q12. When should you prefer a lambda instead?

When the lambda contains additional logic rather than a direct delegation to one existing method.

---

# 25. Common Interview Traps ❌

### Trap 1

`String::length` calls `length()` immediately.

❌ No. It creates method-reference behavior for a compatible target type.

### Trap 2

`Class::method` always means static method.

❌ No. `String::length` is an instance method reference to an arbitrary `String` receiver.

### Trap 3

`object::method` and `Class::method` are identical.

❌ No. The former binds a particular receiver; the latter can use the functional argument as the receiver for an instance method.

### Trap 4

Every lambda can be converted to a method reference.

❌ No. Only when the lambda is effectively a direct method/constructor delegation.

### Trap 5

Method references do not need a functional interface.

❌ A target type is needed to give the method reference its functional context.

---

# 26. Coding Challenges 🔥

### Challenge 1 — Lambda → Method Reference

Convert:

```java
names.forEach(name -> System.out.println(name));
```

to a method reference.

### Challenge 2 — String Transformation

Convert:

```java
Function<String, String> upper =
        text -> text.toUpperCase();
```

to a method reference.

### Challenge 3 — Static Method

Convert:

```java
Function<Integer, Integer> abs =
        number -> Math.abs(number);
```

to a method reference.

### Challenge 4 — Constructor

Convert:

```java
Function<String, User> factory =
        name -> new User(name);
```

to a constructor reference.

### Challenge 5 — Employee Stream ⭐⭐⭐⭐⭐

Given:

```java
employees.stream()
        .map(employee -> employee.getName())
        .forEach(name -> System.out.println(name));
```

Convert both lambdas to method references.

### Challenge 6 — Bound vs Unbound ⭐⭐⭐⭐⭐

Explain the difference between:

```java
text::length
```

and:

```java
String::length
```

### Challenge 7 — Overloaded Method

Create overloaded `print(String)` and `print(Integer)` methods and use appropriate target functional interfaces to resolve each method reference.

---

# 27. 60-Second Interview Answer 🧠

> “A method reference is a concise form of a lambda expression when the lambda directly delegates to an existing method or constructor. Java has four main forms: static method reference like `Math::abs`, particular-object instance method reference like `System.out::println`, arbitrary-object instance method reference like `String::length`, and constructor reference like `User::new`. Method references need a target functional-interface type so the compiler can determine the expected signature. They are heavily used with Streams and improve readability when there is no additional lambda logic.”

---

# 🧪 Practice Code

urlGitHub — 9.5 Method References Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/05-Method-References

---

## Navigation

[← 9.4 — Predicate / Consumer / Supplier / Function](../04-Predicate-Consumer-Supplier-Function/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.5 — Method References → ✅ Completed**

**Next → 9.6 — Optional Fundamentals**