# 10.3 — Local Inner Classes

## 🎯 Interview Goal

Understand **local inner classes**: classes declared inside a method, constructor, or initializer block; their scope, access rules, effectively-final local variables, `this` behavior, and when they are useful compared with lambdas and member inner classes.

---

# 1. What is a Local Inner Class?

A local inner class is a **non-static class declared inside a method, constructor, or initializer block**.

```java
class PaymentService {

    void processPayment() {

        class PaymentValidator {
            void validate() {
                System.out.println("Payment valid");
            }
        }

        PaymentValidator validator = new PaymentValidator();
        validator.validate();
    }
}
```

The class exists only within the scope where it is declared.

### Mental model

```text
Class
  └── method()
        └── Local Inner Class
```

---

# 2. Basic Example ⭐⭐⭐⭐⭐

```java
public class Main {

    static void calculate() {

        class Calculator {

            int add(int a, int b) {
                return a + b;
            }
        }

        Calculator calculator = new Calculator();
        System.out.println(calculator.add(10, 20));
    }

    public static void main(String[] args) {
        calculate();
    }
}
```

Output:

```text
30
```

The `Calculator` type is visible only inside `calculate()`.

---

# 3. Scope of a Local Inner Class

A local inner class cannot normally be referenced outside the block where it is declared.

```java
void process() {

    class Helper {
        void run() {
            System.out.println("Running");
        }
    }

    Helper helper = new Helper();
    helper.run();
}
```

This is invalid outside the method:

```java
// Helper helper = new Helper(); // ❌ Helper is out of scope
```

### Interview line

> “A local inner class has lexical scope limited to the block in which it is declared.”

---

# 4. Why Use a Local Inner Class?

Use one when a helper type:

- is required only by one method/block
- contains more than one related method or piece of state
- needs access to the enclosing instance
- needs to implement/extend a type locally
- should remain hidden from the rest of the class

If the behavior is a single small functional operation, a lambda may often be cleaner.

---

# 5. Local Inner Class Can Access Outer Instance Members ⭐⭐⭐⭐⭐

```java
class OrderService {

    private String serviceName = "OrderService";

    void process() {

        class Processor {
            void print() {
                System.out.println(serviceName);
            }
        }

        new Processor().print();
    }
}
```

A local inner class declared in an instance method can access the enclosing object's instance members.

---

# 6. Local Inner Class Can Access Outer Static Members

```java
class OrderService {

    private static String application = "OrderApp";

    void process() {

        class Processor {
            void print() {
                System.out.println(application);
            }
        }

        new Processor().print();
    }
}
```

It can access enclosing static members as well.

---

# 7. Local Variables Must Be Final or Effectively Final 🔥

This is one of the most important interview topics.

```java
void process() {

    int limit = 10;

    class Validator {
        void validate() {
            System.out.println(limit);
        }
    }

    new Validator().validate();
}
```

This works because `limit` is **effectively final**: it is assigned but never changed.

But this does not compile:

```java
void process() {

    int limit = 10;
    limit = 20;

    class Validator {
        void validate() {
            System.out.println(limit); // ❌
        }
    }
}
```

### Why?

Local variables captured by a local/anonymous inner class must be `final` or **effectively final**.

---

# 8. `final` vs Effectively Final

### Explicit final

```java
final int limit = 10;
```

### Effectively final

```java
int limit = 10;
// never reassigned
```

Both can be captured.

### Not effectively final

```java
int limit = 10;
limit++;
```

Cannot be captured by the local class.

### Interview line ⭐

> “Local variables captured by a local inner class must be final or effectively final.”

---

# 9. Why Does Java Require Effectively Final?

A local variable lives on the method's execution context, while an object of the local class can potentially outlive the method invocation.

Conceptually:

```text
method local variable
        ↓
method execution

local class object
        ↓
may remain reachable
```

Java therefore captures the value in a way that avoids treating the local variable as a shared mutable variable across those lifetimes.

A useful interview phrase is:

> “The captured local is effectively a value captured by the nested object, not a shared mutable local variable.”

---

# 10. Local Inner Class Inside a Static Method

A local class can be declared inside a static method.

```java
class Utility {

    static void execute() {

        class Task {
            void run() {
                System.out.println("Task running");
            }
        }

        new Task().run();
    }
}
```

Because the enclosing method is static, there is no enclosing instance available through that method invocation.

The local class can still access:

- its own members
- accessible static members of the enclosing class
- final/effectively-final local variables

It cannot directly access an enclosing instance field because there is no `this` for the enclosing class in that static context.

---

# 11. Local Inner Class Inside a Constructor

A local class can also be declared inside a constructor.

```java
class Employee {

    Employee(String name) {

        class Initializer {
            void print() {
                System.out.println("Initializing " + name);
            }
        }

        new Initializer().print();
    }
}
```

Here `name` is a constructor parameter and is effectively final.

---

# 12. Local Inner Class Inside an Initializer Block

Local classes can also be declared inside an initializer block.

```java
class Application {

    {
        class StartupTask {
            void run() {
                System.out.println("Startup");
            }
        }

        new StartupTask().run();
    }
}
```

This is less common in production code but useful for understanding the language feature.

---

# 13. Can a Local Inner Class Be `static`? ❌

A local class is not declared as a static member class.

Do not confuse:

```java
class Outer {
    static class Nested { }
}
```

with:

```java
void process() {
    class Local { }
}
```

The first is a **static nested class**. The second is a **local class**.

Modern Java has specific rules around static members in inner classes, but `static class Local` is not the syntax for turning a local class into a static nested class.

---

# 14. Can a Local Inner Class Have a Constructor?

Yes.

```java
void process() {

    class Validator {

        private String type;

        Validator(String type) {
            this.type = type;
        }

        void validate() {
            System.out.println(type + " validated");
        }
    }

    Validator validator = new Validator("Order");
    validator.validate();
}
```

---

# 15. Can a Local Inner Class Implement an Interface? ⭐⭐⭐⭐

Yes.

```java
void execute() {

    class EmailTask implements Runnable {

        @Override
        public void run() {
            System.out.println("Sending email");
        }
    }

    Thread thread = new Thread(new EmailTask());
    thread.start();
}
```

This can be useful when the implementation is needed only locally.

However, for a simple functional interface, a lambda is often more concise.

---

# 16. Local Inner Class Can Extend a Class

```java
void execute() {

    class CustomException extends RuntimeException {

        CustomException(String message) {
            super(message);
        }
    }

    throw new CustomException("Validation failed");
}
```

A local class follows normal inheritance rules.

---

# 17. `this` Inside a Local Inner Class ⭐⭐⭐⭐⭐

```java
class Outer {

    void process() {

        class Local {

            void print() {
                System.out.println(this);
                System.out.println(Outer.this);
            }
        }

        new Local().print();
    }
}
```

Here:

```text
this
 ↓
Local object

Outer.this
 ↓
Outer object
```

This is similar to a member inner class.

---

# 18. Local Inner Class vs Anonymous Inner Class

| Feature | Local Inner Class | Anonymous Inner Class |
|---|---|---|
| Has a class name | ✅ | ❌ |
| Can create multiple objects | ✅ | Usually one-off usage |
| Constructor | ✅ | ❌ explicit named constructor |
| Can implement interface | ✅ | ✅ |
| Can extend class | ✅ | ✅ |
| Reuse within scope | ✅ | Limited |
| Best for | Named local helper | One-off implementation |

Example anonymous class:

```java
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running");
    }
};
```

---

# 19. Local Inner Class vs Lambda ⭐⭐⭐⭐⭐

### Local class

```java
void process() {

    class Validator {
        boolean valid(String value) {
            return value != null && !value.isBlank();
        }
    }

    Validator validator = new Validator();
    System.out.println(validator.valid("Java"));
}
```

### Lambda

```java
Predicate<String> validator =
        value -> value != null && !value.isBlank();
```

### Decision

```text
Need named helper type?
        ↓
       YES → Local class

Single functional behavior?
        ↓
       YES → Lambda often cleaner
```

---

# 20. Important Interview Scenario — Mutable Local Variable

This does **not** compile:

```java
void process() {

    int count = 0;

    class Counter {
        void increment() {
            count++; // ❌
        }
    }
}
```

Why?

`count` is not effectively final.

If mutable state is required, move that state into the local class object:

```java
void process() {

    class Counter {
        private int count;

        void increment() {
            count++;
        }

        int getCount() {
            return count;
        }
    }

    Counter counter = new Counter();
    counter.increment();
    counter.increment();

    System.out.println(counter.getCount());
}
```

This is the cleaner design because the mutable state belongs to the object.

---

# 21. Complete Runnable Practice Code 💻

```java
import java.util.function.Predicate;

public class LocalInnerClassPractice {

    private String serviceName = "OrderService";

    public void processOrders() {

        final int maxOrders = 100;
        String prefix = "ORD-"; // effectively final

        class OrderValidator {

            private int validatedCount;

            boolean validate(String orderId) {
                if (orderId == null || orderId.isBlank()) {
                    return false;
                }

                if (!orderId.startsWith(prefix)) {
                    return false;
                }

                validatedCount++;
                return validatedCount <= maxOrders;
            }

            void printStatus() {
                System.out.println("Service: " + serviceName);
                System.out.println("Validated: " + validatedCount);
            }
        }

        OrderValidator validator = new OrderValidator();

        System.out.println(validator.validate("ORD-101"));
        System.out.println(validator.validate("ORD-102"));
        System.out.println(validator.validate("ABC-103"));

        validator.printStatus();

        // Compare with a lambda for one functional operation.
        Predicate<String> validOrder =
                orderId -> orderId != null && orderId.startsWith(prefix);

        System.out.println(validOrder.test("ORD-200"));
    }

    public static void main(String[] args) {
        LocalInnerClassPractice practice =
                new LocalInnerClassPractice();

        practice.processOrders();
    }
}
```

### Expected output

```text
true
true
false
Service: OrderService
Validated: 2
true
```

---

# 22. Interview Questions 🔥

### Basic

1. What is a local inner class?
2. Where can a local inner class be declared?
3. What is its scope?
4. Can it have a constructor?
5. Can it implement an interface?
6. Can it extend a class?

### Deep Dive

7. Can a local inner class access outer instance fields?
8. Can it access outer static fields?
9. Why must captured local variables be final/effectively final?
10. What does effectively final mean?
11. What does `this` refer to inside a local class?
12. What does `Outer.this` refer to?
13. Can a local class be declared in a static method?
14. Can a local class be declared inside a constructor?
15. Local class vs anonymous class?
16. Local class vs lambda?

### Production / Design

17. When would you use a local inner class?
18. When would you avoid it?
19. How would you handle mutable state that a local class needs?
20. Why might a lambda be preferable for a single functional behavior?

---

# 23. 5-Year Experience Interview Answer 🎤

> “A local inner class is a non-static class declared within a method, constructor, or initializer block. Its scope is limited to that block. I would use it when I need a named helper implementation that is relevant only to one operation, especially when it needs access to the enclosing instance or needs multiple methods/state. Local variables captured by it must be final or effectively final. If the requirement is just one functional behavior, I would generally prefer a lambda or method reference.”

---

# 24. Quick Revision 🧠

```text
Local Inner Class
       ↓
Declared inside method / constructor / block
       ↓
Scope limited to that block
       ↓
Can access enclosing instance members
       ↓
Can access static members
       ↓
Captured local variables
       ↓
final / effectively final
       ↓
Can implement interface / extend class
       ↓
Named helper → Local class
One functional operation → Lambda often preferred
```

### Golden Interview Line ⭐

> “A local inner class is useful when a named helper type is needed only within a specific method or block, while keeping its implementation hidden outside that scope.”

---

# 🧪 Clickable Practice Code

[GitHub — 10.3 Local Inner Classes Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/10-Inner-Classes/10.3-Local-Inner-Classes)

---

## Navigation

[← 10.2 — Static Nested Classes](../10.2-Static-Nested-Classes/README.md)

**Current → 10.3 — Local Inner Classes → ✅**

**Next → 10.4 — Anonymous Inner Classes**