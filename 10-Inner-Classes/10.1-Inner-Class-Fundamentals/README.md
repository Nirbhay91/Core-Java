# Chapter 10 — Inner Classes

## 10.1 — Inner Class Fundamentals

### 🎯 Interview Goal

Understand what an inner class is, why Java supports nested classes, how an inner class accesses the outer object's state, how objects are created, and the interview-level differences between an inner class and a static nested class.

---

# 1. What is an Inner Class?

An **inner class** is a non-static nested class declared inside another class.

```java
class Outer {

    private int value = 100;

    class Inner {
        void show() {
            System.out.println(value);
        }
    }
}
```

Here:

```text
Outer
 └── Inner   ← non-static inner class
```

The important point is that a non-static inner class is associated with an **instance of the outer class**.

---

# 2. Basic Example

```java
class Outer {

    private String message = "Hello from Outer";

    class Inner {

        void display() {
            System.out.println(message);
        }
    }
}

public class Main {

    public static void main(String[] args) {

        Outer outer = new Outer();
        Outer.Inner inner = outer.new Inner();

        inner.display();
    }
}
```

### Output

```text
Hello from Outer
```

### ⭐ Important syntax

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

For a **non-static** inner class, you need an outer-class object before creating the inner-class object.

---

# 3. Why `outer.new Inner()`?

This is one of the most common interview questions.

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

Conceptually:

```text
outer object
    │
    └── associated Inner object
```

The inner object is tied to a particular outer object.

Therefore Java needs to know **which `Outer` instance** the `Inner` instance belongs to.

---

# 4. Inner Class Can Access Outer Instance Members ⭐⭐⭐⭐⭐

```java
class Employee {

    private String name = "Nirbhay";
    private int salary = 150000;

    class EmployeeDetails {

        void printDetails() {
            System.out.println(name);
            System.out.println(salary);
        }
    }
}
```

Even though `name` and `salary` are `private`, the inner class can access them.

### Interview line

> A nested class is allowed to access members of its enclosing class, including private members, subject to the normal Java access rules.

---

# 5. Accessing Outer Object Explicitly — `Outer.this`

```java
class Outer {

    private int value = 10;

    class Inner {

        private int value = 20;

        void print() {
            System.out.println(value);
            System.out.println(this.value);
            System.out.println(Outer.this.value);
        }
    }
}
```

Output:

```text
20
20
10
```

### Remember

```text
this
   ↓
current Inner object

Outer.this
   ↓
associated Outer object
```

🔥 This is a very common interview follow-up.

---

# 6. Inner Class With Constructor

```java
class Car {

    private String model;

    Car(String model) {
        this.model = model;
    }

    class Engine {

        private int power;

        Engine(int power) {
            this.power = power;
        }

        void start() {
            System.out.println(model + " engine: " + power + " HP");
        }
    }
}

public class Main {

    public static void main(String[] args) {
        Car car = new Car("BMW");
        Car.Engine engine = car.new Engine(250);

        engine.start();
    }
}
```

The `Engine` object is associated with the particular `Car` instance.

---

# 7. Multiple Inner Objects Can Belong to Different Outer Objects

```java
class Outer {

    private int value;

    Outer(int value) {
        this.value = value;
    }

    class Inner {
        void print() {
            System.out.println(value);
        }
    }
}

public class Main {

    public static void main(String[] args) {

        Outer first = new Outer(10);
        Outer second = new Outer(20);

        Outer.Inner firstInner = first.new Inner();
        Outer.Inner secondInner = second.new Inner();

        firstInner.print();
        secondInner.print();
    }
}
```

Output:

```text
10
20
```

### Interview insight

The same inner-class type can be associated with different outer-class instances.

---

# 8. Private Inner Class

An inner class can itself be `private`.

```java
class BankAccount {

    private class Security {

        void verify() {
            System.out.println("Verification successful");
        }
    }

    void withdraw() {
        Security security = new Security();
        security.verify();
    }
}
```

The `Security` implementation is hidden from external callers.

### Production use

Useful when the helper implementation:

- belongs tightly to one outer class
- should not be exposed publicly
- needs access to outer state

---

# 9. Inner Class Can Access Static Members Too

```java
class Outer {

    static int count = 100;
    int value = 200;

    class Inner {

        void print() {
            System.out.println(count);
            System.out.println(value);
        }
    }
}
```

A non-static inner class can access both:

```text
Outer static members
Outer instance members
```

because it has an association with an outer instance.

---

# 10. Can an Inner Class Have Static Members?

This is an important modern-Java nuance.

Historically, a non-static inner class could not declare arbitrary static members except constant variables. **Modern Java permits static members in inner classes**, subject to the Java language rules/version in use.

For interview discussions, distinguish:

```text
Static nested class
    ↓
does not require an outer instance

Inner class
    ↓
is non-static and associated with an outer instance
```

Do not incorrectly state as an absolute modern-Java rule that an inner class can never contain static members.

---

# 11. Inner Class vs Static Nested Class ⭐⭐⭐⭐⭐

| Feature | Inner Class | Static Nested Class |
|---|---|---|
| Declared with `static` | ❌ | ✅ |
| Requires outer object | ✅ | ❌ |
| Associated with outer instance | ✅ | ❌ |
| Can directly access outer instance fields | ✅ | ❌ |
| Can access outer static fields | ✅ | ✅ |
| Creation | `outer.new Inner()` | `new Outer.Nested()` |

### Example

```java
class Outer {

    int instanceValue = 10;
    static int staticValue = 20;

    class Inner {
        void print() {
            System.out.println(instanceValue);
            System.out.println(staticValue);
        }
    }

    static class Nested {
        void print() {
            System.out.println(staticValue);
            // instanceValue; // compile-time error
        }
    }
}
```

---

# 12. Why Not Make Every Helper Class Static?

Use a non-static inner class when the helper has a meaningful relationship with a specific outer object and needs direct access to its instance state.

Use a static nested class when that association is unnecessary.

### Design rule

```text
Needs outer instance state?
        │
       YES ──→ Inner class
        │
       NO
        ↓
Static nested class may be better
```

---

# 13. Memory / Lifecycle Interview Concept 🔥

A non-static inner object maintains an association with its enclosing outer instance.

Conceptually:

```text
Outer Object
     ↑
     │ associated with
     │
Inner Object
```

Therefore, if an inner object remains reachable for a long time, the associated outer object may also remain reachable.

### Interview scenario

If a long-lived object stores a non-static inner instance, ask whether that inner instance unintentionally keeps the outer object alive.

This matters especially in:

- caches
- listeners
- callbacks
- long-lived tasks
- application lifecycle objects

---

# 14. Inner Class as Callback / Helper

```java
class Button {

    private String name = "Login";

    class ClickHandler {
        void handle() {
            System.out.println(name + " clicked");
        }
    }

    void click() {
        new ClickHandler().handle();
    }
}
```

The helper directly uses outer state.

In modern Java, a lambda or method reference may often be cleaner when the behavior is functional.

---

# 15. Real Interview Scenario — Why Use Inner Class?

### Question

> Why would you create an inner class instead of a normal top-level class?

### Strong 5-year answer

> “I use an inner class when the helper type is tightly coupled to one enclosing class, especially when it needs direct access to the enclosing instance's state. It also helps encapsulate implementation details. If the helper does not need an outer instance, I would generally consider a static nested class instead because it avoids unnecessary outer-instance association.”

---

# 16. Practice Code — Complete Runnable Example 💻

```java
class Company {

    private String companyName;
    private int employeeCount;

    Company(String companyName, int employeeCount) {
        this.companyName = companyName;
        this.employeeCount = employeeCount;
    }

    class CompanyDetails {

        void print() {
            System.out.println("Company: " + companyName);
            System.out.println("Employees: " + employeeCount);
        }

        void increaseEmployees(int count) {
            employeeCount += count;
        }
    }
}

public class InnerClassFundamentalsPractice {

    public static void main(String[] args) {

        Company company = new Company("TechCorp", 100);

        // Create non-static inner class object
        Company.CompanyDetails details = company.new CompanyDetails();

        details.print();

        details.increaseEmployees(25);

        details.print();

        Company anotherCompany = new Company("Startup", 20);
        Company.CompanyDetails anotherDetails =
                anotherCompany.new CompanyDetails();

        anotherDetails.print();
    }
}
```

### Expected output

```text
Company: TechCorp
Employees: 100
Company: TechCorp
Employees: 125
Company: Startup
Employees: 20
```

---

# 17. Interview Questions 🔥

### Basic

1. What is an inner class?
2. Why is it called non-static nested class?
3. How do you create an inner-class object?
4. Why do we use `outer.new Inner()`?
5. Can an inner class access private outer members?
6. Can an inner class access static members?

### Deep Dive

7. What is `Outer.this`?
8. Inner class vs static nested class?
9. Can different outer objects have different inner objects?
10. Can an inner class have constructors?
11. Can an inner class be private?
12. Can an inner class contain static members in modern Java?
13. How does an inner object relate to the outer object?
14. Can an inner class be inherited?
15. Can an inner class extend another class?
16. Can an inner class implement an interface?

### Design / Production

17. When would you choose an inner class?
18. When would you choose a static nested class?
19. Can an inner object unintentionally keep an outer object reachable?
20. When is a lambda a better choice than an inner class?

---

# 18. Quick Revision 🧠

```text
Inner Class
    ↓
Non-static nested class
    ↓
Associated with Outer instance
    ↓
Creation:
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
    ↓
Can directly access outer instance members
    ↓
`this`       → Inner object
`Outer.this` → Outer object
```

### Golden Interview Line ⭐

> “A non-static inner class is associated with an instance of its enclosing class, so creating it requires an outer-class instance. It can directly access the enclosing instance's members, including private members.”

---

# 🧪 Clickable Practice Code

[GitHub — 10.1 Inner Class Fundamentals Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/10-Inner-Classes/10.1-Inner-Class-Fundamentals)

---

## Navigation

**Chapter 10 — Inner Classes**

**Current → 10.1 — Inner Class Fundamentals → ✅**

**Next → 10.2 — Static Nested Classes**