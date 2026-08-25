# 9.7 — Optional `map()` / `flatMap()`

> **Interview Goal:** Be able to explain exactly why `map()` and `flatMap()` exist, predict their resulting types, avoid nested `Optional`, and solve real repository/domain-object chaining problems.

## 🎯 30-Second Definition

```text
map()     → value → value
flatMap() → value → Optional<value>
```

More precisely:

```java
Optional<T>.map(Function<T, R>)
        → Optional<R>

Optional<T>.flatMap(Function<T, Optional<R>>)
        → Optional<R>
```

### ⭐ Interview line

> Use `map()` when the mapper returns a normal value. Use `flatMap()` when the mapper already returns an `Optional`, so that you don't create `Optional<Optional<T>>`.

---

# 1. Start with a Simple `map()`

```java
Optional<String> name = Optional.of("nirbhay");

Optional<String> upper =
        name.map(String::toUpperCase);

System.out.println(upper.get());
```

Flow:

```text
Optional<String>
      ↓
String::toUpperCase
      ↓
String
      ↓
Optional<String>
```

The mapper returns a normal `String`, so `map()` wraps the result in an `Optional`.

---

# 2. `map()` Signature

The conceptual signature is:

```java
<U> Optional<U> map(Function<? super T, ? extends U> mapper)
```

For interview purposes, remember:

```text
T → R
```

becomes:

```text
Optional<T> → Optional<R>
```

Example:

```java
Optional<String> name = Optional.of("Java");

Optional<Integer> length =
        name.map(String::length);
```

Type transformation:

```text
Optional<String>
        ↓
Function<String, Integer>
        ↓
Optional<Integer>
```

---

# 3. What If `map()` Returns `null`?

This is an important behavior.

```java
Optional<String> result =
        Optional.of("Java")
                .map(value -> null);
```

The result is:

```text
Optional.empty()
```

`Optional.map()` internally does not produce an Optional containing `null`; a null mapping result becomes an empty Optional.

---

# 4. `filter()` + `map()`

A common pipeline is:

```java
Optional<String> name = Optional.of("Nirbhay");

Optional<Integer> result = name
        .filter(value -> value.length() > 5)
        .map(String::length);
```

Flow:

```text
Optional<String>
      ↓
filter → Predicate
      ↓
map    → Function
      ↓
Optional<Integer>
```

---

# 5. The Problem `flatMap()` Solves ⭐⭐⭐⭐⭐

Suppose we have:

```java
class Employee {

    Optional<Address> getAddress() {
        // may or may not exist
    }
}
```

And:

```java
Optional<Employee> employee;
```

If we use `map()`:

```java
Optional<Optional<Address>> result =
        employee.map(Employee::getAddress);
```

Why?

Because:

```text
Employee
   ↓
Employee::getAddress
   ↓
Optional<Address>
```

Then `map()` wraps that result again:

```text
Optional<Optional<Address>>
```

Usually, this nested structure is not what we want.

---

# 6. `flatMap()` Removes the Nesting

Use:

```java
Optional<Address> result =
        employee.flatMap(Employee::getAddress);
```

Flow:

```text
Optional<Employee>
       ↓
Employee::getAddress
       ↓
Optional<Address>
       ↓
flatMap → flatten
       ↓
Optional<Address>
```

### ⭐ Memory

```text
map     → wraps
flatMap → wraps + flattens
```

---

# 7. `map()` vs `flatMap()` — Type-Level Difference

### Case A — mapper returns `R`

```java
Function<Employee, String> mapper =
        Employee::getName;
```

Use:

```java
Optional<String> result =
        employee.map(mapper);
```

### Case B — mapper returns `Optional<R>`

```java
Function<Employee, Optional<Address>> mapper =
        Employee::getAddress;
```

Use:

```java
Optional<Address> result =
        employee.flatMap(mapper);
```

### Interview shortcut

```text
Returns R           → map()
Returns Optional<R> → flatMap()
```

---

# 8. Complete Domain Example ⭐⭐⭐⭐⭐

```java
import java.util.Optional;

class Address {

    private final String city;

    Address(String city) {
        this.city = city;
    }

    public String getCity() {
        return city;
    }
}

class Employee {

    private final String name;
    private final Address address;

    Employee(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public String getName() {
        return name;
    }

    public Optional<Address> getAddress() {
        return Optional.ofNullable(address);
    }
}

public class MapFlatMapDemo {

    public static void main(String[] args) {

        Employee employee =
                new Employee("Nirbhay", new Address("Bangalore"));

        Optional<Employee> optionalEmployee =
                Optional.of(employee);

        // map(): Employee -> String
        Optional<String> name =
                optionalEmployee.map(Employee::getName);

        // flatMap(): Employee -> Optional<Address>
        Optional<Address> address =
                optionalEmployee.flatMap(Employee::getAddress);

        // Continue chaining
        Optional<String> city =
                optionalEmployee
                        .flatMap(Employee::getAddress)
                        .map(Address::getCity);

        System.out.println(name.orElse("Unknown"));
        System.out.println(city.orElse("Unknown City"));
    }
}
```

Expected output:

```text
Nirbhay
Bangalore
```

---

# 9. Multi-Level `flatMap()` ⭐⭐⭐⭐⭐

This is where `flatMap()` becomes very useful in interviews.

Suppose:

```text
Employee
  ↓ Optional<Address>
Address
  ↓ Optional<Country>
Country
  ↓ String
```

Code:

```java
Optional<String> countryName =
        optionalEmployee
                .flatMap(Employee::getAddress)
                .flatMap(Address::getCountry)
                .map(Country::getName);
```

Notice the pattern:

```text
Optional → Optional → Optional → normal value
 flatMap   flatMap      map
```

### Memory

```text
Optional-returning method → flatMap
Normal getter/transformation → map
```

---

# 10. Real Repository-Style Example ⭐⭐⭐⭐⭐

Imagine:

```java
Optional<Employee> findEmployeeById(int id)
```

and:

```java
Optional<Address> findAddress(Employee employee)
```

Instead of:

```java
Optional<Optional<Address>> address =
        findEmployeeById(101)
                .map(this::findAddress);
```

Use:

```java
Optional<Address> address =
        findEmployeeById(101)
                .flatMap(this::findAddress);
```

This is a very common interview scenario.

---

# 11. `map()` with a Normal Getter

```java
Optional<Employee> employee = findEmployeeById(101);

Optional<String> name =
        employee.map(Employee::getName);
```

If `getName()` returns:

```java
String
```

then:

```text
map()
```

is correct.

---

# 12. `flatMap()` with an Optional Getter

If:

```java
Optional<Address> getAddress()
```

then:

```java
employee.flatMap(Employee::getAddress);
```

is correct.

Do **not** unnecessarily do:

```java
employee.map(Employee::getAddress)
```

because that creates:

```java
Optional<Optional<Address>>
```

---

# 13. `map()` vs `flatMap()` with Method References

### `map`

```java
optionalEmployee.map(Employee::getName);
```

Because:

```java
Employee::getName
```

returns:

```text
String
```

### `flatMap`

```java
optionalEmployee.flatMap(Employee::getAddress);
```

Because:

```java
Employee::getAddress
```

returns:

```text
Optional<Address>
```

This connects directly with **9.5 Method References**.

---

# 14. `map()` / `flatMap()` + `orElseGet()`

A production-style pipeline:

```java
String city = optionalEmployee
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .orElseGet(() -> "Unknown City");
```

Functional interface mapping:

```text
flatMap()   → Function returning Optional
map()       → Function
orElseGet() → Supplier
```

---

# 15. `map()` / `flatMap()` + `filter()`

```java
Optional<String> city = optionalEmployee
        .filter(employee -> employee.getName() != null)
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .filter(value -> value.equalsIgnoreCase("Bangalore"));
```

Flow:

```text
Optional<Employee>
       ↓
filter → Predicate
       ↓
flatMap → Optional<Address>
       ↓
map → Optional<String>
       ↓
filter → Predicate
```

---

# 16. `Optional<Optional<T>>` — Interview Trap 🔥

Question:

> Why is `Optional<Optional<Address>>` usually undesirable?

Answer:

> Because it creates two levels of absence that the caller must unwrap separately. `flatMap()` flattens the nested Optional into one `Optional<Address>` and makes the pipeline easier to compose.

Example:

```java
Optional<Optional<Address>> nested =
        optionalEmployee.map(Employee::getAddress);
```

Flatten it manually:

```java
Optional<Address> flattened =
        nested.flatMap(value -> value);
```

But the cleaner approach is:

```java
Optional<Address> address =
        optionalEmployee.flatMap(Employee::getAddress);
```

---

# 17. `flatMap()` Is Not Just “Remove One Optional”

A better interview explanation:

`flatMap()` is designed for composing operations where each operation may itself produce an Optional.

Think:

```text
T → Optional<R>
```

When chaining from:

```text
Optional<T>
```

`flatMap()` lets the next optional-producing operation participate directly in the same pipeline.

---

# 18. Null Handling in `map()`

Consider:

```java
Optional<String> result =
        Optional.of("Java")
                .map(value -> null);
```

Result:

```text
Optional.empty()
```

This is different from an Optional mapper that is itself returning an Optional.

If your mapper already returns Optional, use `flatMap()`.

---

# 19. `flatMap()` Mapper Should Not Return Null

If you use:

```java
optional.flatMap(value -> null);
```

the Optional API cannot treat that as a valid Optional result. The mapper is expected to return a non-null `Optional`.

Return:

```java
Optional.empty()
```

instead of null when there is no result.

Example:

```java
optional.flatMap(value ->
        value.length() > 5
                ? Optional.of(value)
                : Optional.empty());
```

---

# 20. Complete Interview Practice Program ⭐⭐⭐⭐⭐

```java
import java.util.Optional;

class Country {

    private final String name;

    Country(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

class Address {

    private final String city;
    private final Country country;

    Address(String city, Country country) {
        this.city = city;
        this.country = country;
    }

    public String getCity() {
        return city;
    }

    public Optional<Country> getCountry() {
        return Optional.ofNullable(country);
    }
}

class Employee {

    private final String name;
    private final Address address;

    Employee(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public String getName() {
        return name;
    }

    public Optional<Address> getAddress() {
        return Optional.ofNullable(address);
    }
}

public class OptionalChainingInterviewDemo {

    public static void main(String[] args) {

        Country country =
                new Country("India");

        Address address =
                new Address("Bangalore", country);

        Employee employee =
                new Employee("Nirbhay", address);

        Optional<Employee> optionalEmployee =
                Optional.of(employee);

        // Normal transformation → map
        Optional<String> employeeName =
                optionalEmployee.map(Employee::getName);

        // Optional-returning transformation → flatMap
        Optional<Address> employeeAddress =
                optionalEmployee.flatMap(Employee::getAddress);

        // Multi-level Optional chaining
        Optional<String> countryName =
                optionalEmployee
                        .flatMap(Employee::getAddress)
                        .flatMap(Address::getCountry)
                        .map(Country::getName);

        // Complete pipeline
        String city = optionalEmployee
                .flatMap(Employee::getAddress)
                .map(Address::getCity)
                .orElseGet(() -> "Unknown City");

        System.out.println("Employee: " +
                employeeName.orElse("Unknown"));

        System.out.println("Address present: " +
                employeeAddress.isPresent());

        System.out.println("Country: " +
                countryName.orElse("Unknown Country"));

        System.out.println("City: " + city);
    }
}
```

Expected output:

```text
Employee: Nirbhay
Address present: true
Country: India
City: Bangalore
```

---

# 21. Practice with Missing Data

Change:

```java
Employee employee =
        new Employee("Nirbhay", null);
```

Then:

```java
String city = optionalEmployee
        .flatMap(Employee::getAddress)
        .map(Address::getCity)
        .orElse("Unknown City");
```

Result:

```text
Unknown City
```

No explicit nested null checks are required in the pipeline.

---

# 22. Interview Scenario — Nested Object Lookup ⭐⭐⭐⭐⭐

Question:

> An Employee may have an Address, and an Address may have a Country. How would you safely get the country name?

Answer:

```java
String country = employeeOptional
        .flatMap(Employee::getAddress)
        .flatMap(Address::getCountry)
        .map(Country::getName)
        .orElse("Unknown");
```

Explain:

```text
Employee → Optional<Address> → flatMap
Address  → Optional<Country> → flatMap
Country  → String            → map
```

This is a strong 5-year Java interview example.

---

# 23. Interview Scenario — Repository Chaining

Given:

```java
Optional<User> findUser(long id)
Optional<Account> findAccount(User user)
Optional<Address> findAddress(Account account)
```

Use:

```java
Optional<Address> address =
        findUser(101)
                .flatMap(this::findAccount)
                .flatMap(this::findAddress);
```

Why `flatMap()` every time?

Because each next method returns an `Optional`.

If the final operation returns a normal value:

```java
Optional<String> city =
        findUser(101)
                .flatMap(this::findAccount)
                .flatMap(this::findAddress)
                .map(Address::getCity);
```

---

# 24. `map()` vs `flatMap()` Cheat Sheet 🧠

| Mapper returns | Use | Result |
|---|---|---|
| `String` | `map()` | `Optional<String>` |
| `Integer` | `map()` | `Optional<Integer>` |
| `EmployeeDTO` | `map()` | `Optional<EmployeeDTO>` |
| `Optional<Address>` | `flatMap()` | `Optional<Address>` |
| `Optional<Account>` | `flatMap()` | `Optional<Account>` |
| `Optional<String>` | `flatMap()` | `Optional<String>` |

### ⭐ One-line rule

```text
Normal value       → map()
Optional value     → flatMap()
```

---

# 25. Interview Questions 🔥

### Q1. What is `map()` in Optional?

It transforms the value inside an Optional using a `Function` and returns an Optional containing the transformed result.

### Q2. What is `flatMap()`?

It is used when the mapper itself returns an Optional, allowing optional-producing operations to be chained without creating nested Optionals.

### Q3. What happens with `optional.map(x -> Optional.of(x))`?

It produces:

```java
Optional<Optional<T>>
```

### Q4. How do you avoid `Optional<Optional<T>>`?

Use `flatMap()`.

### Q5. What does `map()` return if its mapper returns null?

An empty Optional.

### Q6. What should a `flatMap()` mapper return when there is no value?

`Optional.empty()`, not null.

### Q7. Give a real-world use case for `flatMap()`.

Chaining repository/domain operations where each lookup may return an Optional, such as User → Account → Address.

### Q8. Can `map()` and `flatMap()` be mixed?

Yes, and that is common. Use `flatMap()` for Optional-returning operations and `map()` for normal transformations.

### Q9. Why not just call `get()` between operations?

Because it throws when the value is absent and breaks the safe fluent pipeline.

### Q10. Is `flatMap()` only available on Optional?

No. Java Streams also have `flatMap()`, but its purpose there is flattening nested streams rather than nested Optionals.

### Q11. Which functional interface is used by both methods?

Both accept a `Function`; the difference is the expected return shape of the function.

### Q12. Is `flatMap()` always better than `map()`?

No. Use `map()` when the mapper returns a normal value. `flatMap()` is appropriate when the mapper already returns an Optional.

---

# 26. Common Interview Traps ❌

### Trap 1

> `flatMap()` is always faster than `map()`.

❌ No. They solve different type/composition problems.

### Trap 2

> Use `flatMap()` for every transformation.

❌ No. Normal value → `map()`.

### Trap 3

> `map()` always creates `Optional<Optional<T>>`.

❌ Only when the mapper itself returns an Optional.

### Trap 4

> `flatMap()` can return null.

❌ The mapper should return a non-null Optional, usually `Optional.empty()` when absent.

### Trap 5

> `map()` unwraps an Optional returned by the mapper.

❌ `map()` wraps the mapper's result; `flatMap()` handles Optional-returning mappers without nesting.

### Trap 6

> `flatMap()` eliminates every null in the application.

❌ It only provides Optional-based composition where your API actually uses Optional.

---

# 27. Coding Challenges 🔥

### Challenge 1 — Basic `map()`

Given:

```java
Optional<String> name = Optional.of("Nirbhay");
```

convert it to its length using `map()`.

Expected type:

```java
Optional<Integer>
```

### Challenge 2 — Basic `flatMap()`

Create:

```java
Optional<Address> getAddress()
```

inside Employee and safely retrieve it from `Optional<Employee>`.

### Challenge 3 — Nested Optional

Create code that produces:

```java
Optional<Optional<String>>
```

using `map()`, then rewrite it correctly using `flatMap()`.

### Challenge 4 — Employee → Address → City ⭐⭐⭐⭐⭐

Implement:

```text
Optional<Employee>
        ↓
Optional<Address>
        ↓
String city
```

using `flatMap()` and `map()`.

### Challenge 5 — Employee → Account → Country ⭐⭐⭐⭐⭐

Every lookup returns Optional. Chain all operations without calling `get()`.

### Challenge 6 — Missing Data

Test your pipeline when:

```text
Employee missing
Address missing
Country missing
```

The result should safely become:

```text
"Unknown"
```

### Challenge 7 — Interview Explanation

Explain this line without running it:

```java
String city = employee
        .flatMap(Employee::getAddress)
        .flatMap(Address::getCountry)
        .map(Country::getName)
        .orElse("Unknown");
```

Expected explanation:

```text
Employee → Optional<Address> → flatMap
Address  → Optional<Country> → flatMap
Country  → String            → map
absence  → "Unknown"         → orElse
```

---

# 28. 60-Second Interview Answer 🧠

> “In Optional, `map()` is used when my transformation returns a normal value. For example, `Optional<Employee>.map(Employee::getName)` produces `Optional<String>`. `flatMap()` is used when the transformation itself returns an Optional, such as `Employee::getAddress` returning `Optional<Address>`. If I used `map()` there, I would get `Optional<Optional<Address>>`. `flatMap()` prevents that nesting and lets me compose multiple optional-returning operations, such as Employee → Address → Country, while normal transformations at the end can use `map()`.”

---

# 🧠 Final Revision Sheet

```text
┌───────────────────────────────────────────┐
│ map()                                     │
│                                           │
│ Optional<T> + Function<T,R>               │
│              ↓                            │
│         Optional<R>                       │
└───────────────────────────────────────────┘

┌───────────────────────────────────────────┐
│ flatMap()                                  │
│                                            │
│ Optional<T> + Function<T,Optional<R>>      │
│              ↓                             │
│         Optional<R>                        │
└───────────────────────────────────────────┘
```

### Golden Rule ⭐

```text
Mapper returns R
      → map()

Mapper returns Optional<R>
      → flatMap()
```

### Real-world chain

```java
String country = employeeOptional
        .flatMap(Employee::getAddress)
        .flatMap(Address::getCountry)
        .map(Country::getName)
        .orElse("Unknown");
```

Remember:

```text
Optional-returning operation → flatMap
Normal transformation        → map
Fallback                     → orElse / orElseGet
```

---

# 🧪 Practice Code

urlGitHub — 9.7 Optional map / flatMap Practice Codehttps://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/07-Optional-map-flatMap

---

## Navigation

[← 9.6 — Optional Fundamentals](../06-Optional-Fundamentals/README.md)

[🏠 Chapter 9 — Java 8 Modern Features](../README.md)

**Current → 9.7 — Optional `map()` / `flatMap()` → ✅ Completed**

**Next → 9.8 — Optional `filter()` / `orElse()` / `orElseGet()`**