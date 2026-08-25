# 10.2 — Static Nested Classes

## 🎯 Interview Goal

Understand **static nested classes**, how they differ from non-static inner classes, how they are instantiated, what they can access, and when they are the better design choice.

---

# 1. What is a Static Nested Class?

A static nested class is a class declared inside another class using `static`.

```java
class Outer {

    static class Nested {
        void show() {
            System.out.println("Static nested class");
        }
    }
}
```

Unlike a non-static inner class, a static nested class **does not require an instance of the outer class**.

```java
Outer.Nested nested = new Outer.Nested();
nested.show();
```

### Mental model

```text
Outer
 └── static Nested

No Outer object required
```

---

# 2. Basic Example ⭐⭐⭐⭐⭐

```java
class Company {

    static String companyName = "TechCorp";

    static class CompanyDetails {

        void print() {
            System.out.println(companyName);
        }
    }
}

public class Main {

    public static void main(String[] args) {
        Company.CompanyDetails details = new Company.CompanyDetails();
        details.print();
    }
}
```

Output:

```text
TechCorp
```

### Important syntax

```java
Outer.Nested object = new Outer.Nested();
```

There is **no**:

```java
outer.new Nested(); // ❌
```

because the nested class is static.

---

# 3. Why Doesn't It Need an Outer Object?

A static nested class belongs to the **enclosing class**, not to a particular enclosing object.

```text
Non-static Inner Class

Outer Object A ──→ Inner Object A
Outer Object B ──→ Inner Object B


Static Nested Class

Outer Class ──→ Nested Class
```

The nested class has no implicit association with an outer instance.

---

# 4. Accessing Outer Static Members

A static nested class can directly access static members of the outer class.

```java
class Outer {

    private static int count = 100;

    static class Nested {

        void print() {
            System.out.println(count);
        }
    }
}
```

This works because `count` belongs to `Outer` itself.

---

# 5. Can Static Nested Class Access Outer Instance Members? 🔥

No, not directly.

```java
class Outer {

    private int value = 100;

    static class Nested {

        void print() {
            // System.out.println(value); // ❌ compile-time error
        }
    }
}
```

Why?

```text
value
 ↓
belongs to an Outer object

Nested
 ↓
has no Outer object association
```

### Interview line

> “A static nested class can directly access static members of the enclosing class, but it cannot directly access enclosing instance members because it has no implicit enclosing instance.”

---

# 6. Access Instance Members by Passing an Object

Although it cannot access instance state implicitly, you can explicitly pass an outer object.

```java
class Outer {

    private int value = 100;

    static class Nested {

        void print(Outer outer) {
            System.out.println(outer.value);
        }
    }
}

public class Main {

    public static void main(String[] args) {
        Outer outer = new Outer();
        Outer.Nested nested = new Outer.Nested();

        nested.print(outer);
    }
}
```

This is ordinary object access—not an implicit inner-class relationship.

---

# 7. `Outer.this` Is Not Available

A static nested class has no enclosing instance.

Therefore this is invalid:

```java
class Outer {

    static class Nested {
        void print() {
            // System.out.println(Outer.this); // ❌
        }
    }
}
```

Compare:

```text
Inner class:
Outer.this   ✅

Static nested class:
Outer.this   ❌
```

---

# 8. Static Nested Class Can Have Static and Instance Members

```java
class Outer {

    static class Nested {

        static int staticValue = 10;
        int instanceValue = 20;

        void print() {
            System.out.println(staticValue);
            System.out.println(instanceValue);
        }
    }
}
```

The nested class is a normal class in this respect. It can have its own:

- instance fields
- static fields
- constructors
- methods
- static methods
- nested types

subject to the normal Java language rules.

---

# 9. Constructor in Static Nested Class

```java
class Database {

    static class ConnectionConfig {

        private String url;

        ConnectionConfig(String url) {
            this.url = url;
        }

        void print() {
            System.out.println("URL: " + url);
        }
    }
}

public class Main {

    public static void main(String[] args) {
        Database.ConnectionConfig config =
                new Database.ConnectionConfig("jdbc:mysql://localhost/app");

        config.print();
    }
}
```

No `Database` object is required.

---

# 10. Static Nested Class vs Inner Class ⭐⭐⭐⭐⭐

| Feature | Non-static Inner Class | Static Nested Class |
|---|---|---|
| `static` declaration | ❌ | ✅ |
| Requires outer object | ✅ | ❌ |
| Implicit outer reference | ✅ | ❌ |
| Direct outer instance access | ✅ | ❌ |
| Direct outer static access | ✅ | ✅ |
| `Outer.this` | ✅ | ❌ |
| Creation | `outer.new Inner()` | `new Outer.Nested()` |
| Good for | Object-specific helper | Class-level helper / encapsulated type |

### Interview shortcut

```text
Needs enclosing object state?
        ↓
       YES → Inner class
        ↓
       NO  → Consider static nested class
```

---

# 11. Why Prefer Static Nested Class When Possible? 🔥

If a nested helper does not need the outer instance, making it static makes the relationship explicit.

Benefits:

- no implicit outer-instance association
- clearer ownership
- easier to reason about lifecycle
- avoids unnecessary outer reference
- can improve encapsulation
- can be instantiated independently of an outer object

### Interview line

> “If the nested type does not need enclosing instance state, I generally prefer a static nested class because it expresses that dependency explicitly and avoids an unnecessary outer-instance association.”

---

# 12. Real-World Example — Builder Pattern ⭐⭐⭐⭐⭐

One of the most common practical examples is the Builder pattern.

```java
class Employee {

    private final int id;
    private final String name;
    private final String department;

    private Employee(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.department = builder.department;
    }

    static class Builder {

        private int id;
        private String name;
        private String department;

        Builder id(int id) {
            this.id = id;
            return this;
        }

        Builder name(String name) {
            this.name = name;
            return this;
        }

        Builder department(String department) {
            this.department = department;
            return this;
        }

        Employee build() {
            return new Employee(this);
        }
    }
}
```

Usage:

```java
Employee employee = new Employee.Builder()
        .id(101)
        .name("Nirbhay")
        .department("IT")
        .build();
```

### Why static?

The builder does not need a particular `Employee` instance. It is used to construct one.

---

# 13. Real-World Example — Grouping Configuration

```java
class Server {

    static class Config {
        final String host;
        final int port;

        Config(String host, int port) {
            this.host = host;
            this.port = port;
        }
    }
}
```

Usage:

```java
Server.Config config = new Server.Config("localhost", 8080);
```

This keeps `Config` conceptually grouped under `Server` without tying it to a particular `Server` object.

---

# 14. Access Modifiers

A static nested class can be declared with normal access modifiers when declared as a member type:

```java
class Outer {

    public static class PublicNested { }

    protected static class ProtectedNested { }

    private static class PrivateNested { }

    static class PackageNested { }
}
```

This makes nested classes useful for encapsulating implementation details.

---

# 15. Private Static Nested Class — Encapsulation

```java
public class PaymentService {

    public void process() {
        PaymentValidator validator = new PaymentValidator();
        validator.validate();
    }

    private static class PaymentValidator {

        void validate() {
            System.out.println("Payment validated");
        }
    }
}
```

External code cannot directly access `PaymentValidator`.

This is useful when the helper type is an implementation detail of the enclosing class.

---

# 16. Static Nested Class and Memory ⭐⭐⭐⭐

A static nested class does **not** carry an implicit reference to an outer object merely because it is nested.

Compare:

```text
Inner object
    ↓
implicit association with Outer instance

Static nested object
    ↓
no implicit Outer instance association
```

This is an important reason a static nested class can be preferable when outer state is not required.

---

# 17. Common Interview Trap ❌

### Question

> Is a static nested class the same as a top-level class?

### Answer

Not exactly.

A static nested class:

- is still declared inside another class
- has a qualified name such as `Outer.Nested`
- participates in the enclosing class's access-control structure
- can be private/protected/package-private/public as a member type
- is not associated with an enclosing instance

---

# 18. Common Interview Trap — `static` Meaning

Do not say:

> “Static nested class means only one object can exist.” ❌

You can create many instances:

```java
Outer.Nested first = new Outer.Nested();
Outer.Nested second = new Outer.Nested();
```

`static` here means the **nested type does not require an enclosing object**, not that its instances are singleton.

---

# 19. Complete Runnable Practice Code 💻

```java
public class StaticNestedClassPractice {

    static class Company {

        private static String companyName = "TechCorp";
        private int employeeCount;

        Company(int employeeCount) {
            this.employeeCount = employeeCount;
        }

        static class CompanyDetails {

            private String department;

            CompanyDetails(String department) {
                this.department = department;
            }

            void printCompany() {
                System.out.println("Company: " + companyName);
                System.out.println("Department: " + department);
            }

            void printEmployeeCount(Company company) {
                System.out.println("Employees: " + company.employeeCount);
            }
        }
    }

    public static void main(String[] args) {

        // No Company object required to create CompanyDetails.
        Company.CompanyDetails details =
                new Company.CompanyDetails("Engineering");

        details.printCompany();

        // Explicitly pass a Company object when instance state is needed.
        Company company = new Company(150);
        details.printEmployeeCount(company);

        // Multiple nested-class objects are allowed.
        Company.CompanyDetails hrDetails =
                new Company.CompanyDetails("HR");

        hrDetails.printCompany();
    }
}
```

### Expected output

```text
Company: TechCorp
Department: Engineering
Employees: 150
Company: TechCorp
Department: HR
```

---

# 20. Interview Scenarios 🔥

### Scenario 1

**Question:** You have a helper class inside `OrderService`, but it does not need any `OrderService` instance fields. What would you choose?

**Answer:** Consider a `static` nested class.

---

### Scenario 2

**Question:** Why can't a static nested class directly access an outer instance field?

**Answer:** Because there is no implicit enclosing instance associated with the static nested class.

---

### Scenario 3

**Question:** Can you create a static nested class without creating the outer class object?

**Answer:** Yes.

```java
Outer.Nested nested = new Outer.Nested();
```

---

### Scenario 4

**Question:** Can a static nested class access private static members of the outer class?

**Answer:** Yes, subject to normal Java access rules.

---

### Scenario 5

**Question:** Why is `Builder` often static nested?

**Answer:** The builder constructs an outer object; it does not need an existing outer object instance.

---

# 21. 5-Year Experience Interview Answer 🎤

> “A static nested class is a member class declared with `static`. Unlike a non-static inner class, it has no implicit reference to an enclosing instance, so I can instantiate it using `Outer.Nested` without creating an `Outer` object. It can directly access the enclosing class's static members but not its instance members. I prefer it when the nested type is conceptually part of the outer class but doesn't depend on a particular outer object. A common example is a Builder or configuration/helper type.”

---

# 22. Quick Revision 🧠

```text
Static Nested Class
        ↓
Declared using static
        ↓
No Outer object required
        ↓
new Outer.Nested()
        ↓
No implicit Outer reference
        ↓
Can directly access Outer static members
        ↓
Cannot directly access Outer instance members
        ↓
Useful for Builder / Config / Helper / Encapsulation
```

### Golden Interview Line ⭐

> “Static nested means nested inside the outer class but independent of any particular outer-class instance.”

---

# 🧪 Clickable Practice Code

[GitHub — 10.2 Static Nested Classes Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/10-Inner-Classes/10.2-Static-Nested-Classes)

---

## Navigation

[← 10.1 — Inner Class Fundamentals](../10.1-Inner-Class-Fundamentals/README.md)

**Current → 10.2 — Static Nested Classes → ✅**

**Next → 10.3 — Local Inner Classes**