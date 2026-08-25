# 9.16 — `Collectors.toMap()` / `toSet()` / `toList()` Deep Dive

## 🎯 Interview Goal

Master the three collection patterns used constantly in Java Stream coding:

```text
toList() → Stream → List
toSet()  → Stream → Set
toMap()  → Stream → Map
```

The most important interview trap is **duplicate keys with `toMap()`**.

---

# 1. `toList()` Fundamentals ⭐⭐⭐⭐⭐

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toList());
```

Mental model:

```text
Stream
  ↓
collect(toList())
  ↓
List
```

Use it when you need to materialize stream elements into a List.

> Modern JDKs also provide `Stream.toList()`. `Collectors.toList()` and `Stream.toList()` are related but not identical APIs; do not claim that `Collectors.toList()` guarantees a particular mutability or concrete List implementation.

---

# 2. `Stream.toList()` vs `Collectors.toList()` 🔥🔥🔥

```java
List<String> a = stream.collect(Collectors.toList());
```

versus:

```java
List<String> b = stream.toList();
```

Important interview point:

```text
Stream.toList()
→ returns an unmodifiable List

Collectors.toList()
→ does not guarantee mutability, implementation or exact List type
```

If you explicitly need a mutable list:

```java
List<String> names = stream.collect(Collectors.toCollection(ArrayList::new));
```

---

# 3. `toSet()` Fundamentals ⭐⭐⭐⭐⭐

```java
Set<String> departments = employees.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toSet());
```

Use it when duplicate values should be removed.

```text
IT
IT
HR
Finance
IT
```

becomes conceptually:

```text
IT
HR
Finance
```

Do not assume ordering from `Collectors.toSet()`.

---

# 4. `toCollection()` When You Need a Specific Set/List

For `LinkedHashSet`:

```java
Set<String> departments = employees.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toCollection(LinkedHashSet::new));
```

For `TreeSet`:

```java
Set<String> departments = employees.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toCollection(TreeSet::new));
```

For `ArrayList`:

```java
List<String> names = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.toCollection(ArrayList::new));
```

### Interview line

> `toCollection()` is useful when I need control over the concrete collection implementation.

---

# 5. `toMap()` Fundamentals ⭐⭐⭐⭐⭐

Create employee ID → employee map:

```java
Map<Integer, Employee> employeesById = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity()
        ));
```

Mental model:

```text
Stream<Employee>
      ↓
key mapper + value mapper
      ↓
Map<Integer, Employee>
```

---

# 6. `toMap()` Signature 🔥

Common overloads:

```java
Collectors.toMap(keyMapper, valueMapper)
```

```java
Collectors.toMap(keyMapper, valueMapper, mergeFunction)
```

```java
Collectors.toMap(keyMapper, valueMapper, mergeFunction, mapFactory)
```

The third argument is critical when duplicate keys are possible.

---

# 7. Duplicate Key Trap ⭐⭐⭐⭐⭐

This can fail if two employees have the same department:

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity()
        ));
```

The two-argument `toMap()` throws `IllegalStateException` when duplicate keys are encountered.

### Fix with merge function

Keep the first employee:

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                (existing, replacement) -> existing
        ));
```

Keep the latest employee:

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                (existing, replacement) -> replacement
        ));
```

---

# 8. `toMap()` With Numeric Values

Employee name → salary:

```java
Map<String, Integer> salaryMap = employees.stream()
        .collect(Collectors.toMap(
                Employee::getName,
                Employee::getSalary
        ));
```

This assumes employee names are unique.

If they are not unique, use an appropriate merge strategy or choose a different key.

---

# 9. `toMap()` With Transformation

Name → uppercase name:

```java
Map<Integer, String> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                e -> e.getName().toUpperCase(Locale.ROOT)
        ));
```

---

# 10. `toMap()` + Merge Function — Sum Values 🔥🔥🔥

If duplicate product IDs occur, sum their quantities:

```java
Map<Integer, Integer> quantityByProduct = products.stream()
        .collect(Collectors.toMap(
                Product::getId,
                Product::getQuantity,
                Integer::sum
        ));
```

Mental model:

```text
same key?
   ↓
merge old value + new value
```

---

# 11. `toMap()` + Merge Function — Maximum Value

```java
Map<String, Employee> highestPaidByDepartment = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                BinaryOperator.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

This is an excellent interview pattern.

---

# 12. `toMap()` + `LinkedHashMap` 🔥

If you need insertion-order map behavior:

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity(),
                (a, b) -> a,
                LinkedHashMap::new
        ));
```

The fourth argument is the map factory.

---

# 13. `toMap()` + `TreeMap`

If you need sorted keys:

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity(),
                (a, b) -> a,
                TreeMap::new
        ));
```

Now the resulting map uses `TreeMap` ordering by key.

---

# 14. `toConcurrentMap()` ⭐⭐⭐⭐⭐

For concurrent map collection:

```java
ConcurrentMap<Integer, Employee> result = employees.parallelStream()
        .collect(Collectors.toConcurrentMap(
                Employee::getId,
                Function.identity()
        ));
```

Duplicate keys still require an appropriate merge function.

---

# 15. `toSet()` vs `toList()` 🔥

| Requirement | Collector |
|---|---|
| Preserve duplicates | `toList()` |
| Remove duplicates | `toSet()` |
| Need insertion-order Set | `toCollection(LinkedHashSet::new)` |
| Need sorted Set | `toCollection(TreeSet::new)` |
| Need specific List implementation | `toCollection(ArrayList::new)` |

Important:

> `toSet()` does not mean sorted Set.

---

# 16. `toMap()` vs `groupingBy()` ⭐⭐⭐⭐⭐

### One value per key

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity()
        ));
```

### Multiple values per key

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

### Interview answer

> I use `toMap()` when the result is naturally one value per key, with a merge function if duplicate keys are valid. I use `groupingBy()` when one key can naturally contain multiple elements or when I need downstream aggregation.

---

# 17. `toMap()` vs `partitioningBy()`

`toMap()` creates arbitrary keys from a key-mapping function:

```java
Collectors.toMap(Employee::getId, Function.identity())
```

`partitioningBy()` creates boolean partitions:

```java
Collectors.partitioningBy(e -> e.getSalary() > 100000)
```

---

# 18. Complete Runnable Practice Code ⭐⭐⭐⭐⭐

```java
import java.util.*;
import java.util.concurrent.ConcurrentMap;
import java.util.function.BinaryOperator;
import java.util.function.Function;
import java.util.stream.Collectors;

public class ToMapToSetToListDemo {

    static class Employee {
        private final int id;
        private final String name;
        private final String department;
        private final int salary;

        Employee(int id, String name, String department, int salary) {
            this.id = id;
            this.name = name;
            this.department = department;
            this.salary = salary;
        }

        public int getId() {
            return id;
        }

        public String getName() {
            return name;
        }

        public String getDepartment() {
            return department;
        }

        public int getSalary() {
            return salary;
        }

        @Override
        public String toString() {
            return id + " - " + name + " - " + department + " - " + salary;
        }
    }

    static class Product {
        private final int id;
        private final String name;
        private final int quantity;

        Product(int id, String name, int quantity) {
            this.id = id;
            this.name = name;
            this.quantity = quantity;
        }

        public int getId() {
            return id;
        }

        public String getName() {
            return name;
        }

        public int getQuantity() {
            return quantity;
        }
    }

    public static void main(String[] args) {

        List<Employee> employees = Arrays.asList(
                new Employee(1, "Nirbhay", "IT", 150000),
                new Employee(2, "Rahul", "IT", 120000),
                new Employee(3, "Amit", "IT", 180000),
                new Employee(4, "Priya", "HR", 130000),
                new Employee(5, "Ravi", "HR", 90000),
                new Employee(6, "Sneha", "Finance", 110000)
        );

        // 1. Stream -> List
        List<String> names = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.toList());

        // 2. Stream -> Set
        Set<String> departments = employees.stream()
                .map(Employee::getDepartment)
                .collect(Collectors.toSet());

        // 3. Stream -> insertion-order Set
        Set<String> orderedDepartments = employees.stream()
                .map(Employee::getDepartment)
                .collect(Collectors.toCollection(LinkedHashSet::new));

        // 4. Stream -> sorted Set
        Set<String> sortedDepartments = employees.stream()
                .map(Employee::getDepartment)
                .collect(Collectors.toCollection(TreeSet::new));

        // 5. ID -> Employee
        Map<Integer, Employee> byId = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getId,
                        Function.identity()
                ));

        // 6. Name -> Salary
        Map<String, Integer> salaryByName = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getName,
                        Employee::getSalary
                ));

        // 7. Department -> first employee
        Map<String, Employee> firstEmployeeByDepartment = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getDepartment,
                        Function.identity(),
                        (existing, replacement) -> existing
                ));

        // 8. Department -> latest employee
        Map<String, Employee> latestEmployeeByDepartment = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getDepartment,
                        Function.identity(),
                        (existing, replacement) -> replacement
                ));

        // 9. Department -> highest-paid employee
        Map<String, Employee> highestPaidByDepartment = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getDepartment,
                        Function.identity(),
                        BinaryOperator.maxBy(
                                Comparator.comparingInt(Employee::getSalary)
                        )
                ));

        // 10. ID -> Employee using LinkedHashMap
        Map<Integer, Employee> linkedMap = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getId,
                        Function.identity(),
                        (a, b) -> a,
                        LinkedHashMap::new
                ));

        // 11. ID -> Employee using TreeMap
        Map<Integer, Employee> treeMap = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getId,
                        Function.identity(),
                        (a, b) -> a,
                        TreeMap::new
                ));

        // 12. Duplicate product IDs -> sum quantity
        List<Product> products = Arrays.asList(
                new Product(101, "Laptop", 2),
                new Product(102, "Mouse", 5),
                new Product(101, "Laptop", 3),
                new Product(102, "Mouse", 2)
        );

        Map<Integer, Integer> quantityByProduct = products.stream()
                .collect(Collectors.toMap(
                        Product::getId,
                        Product::getQuantity,
                        Integer::sum
                ));

        // 13. Names as uppercase values
        Map<Integer, String> upperCaseNames = employees.stream()
                .collect(Collectors.toMap(
                        Employee::getId,
                        e -> e.getName().toUpperCase(Locale.ROOT)
                ));

        // 14. Concurrent map
        ConcurrentMap<Integer, Employee> concurrentMap = employees.parallelStream()
                .collect(Collectors.toConcurrentMap(
                        Employee::getId,
                        Function.identity()
                ));

        // 15. Modern Stream.toList()
        List<String> immutableNames = employees.stream()
                .map(Employee::getName)
                .toList();

        System.out.println("Names: " + names);
        System.out.println("Departments: " + departments);
        System.out.println("Ordered departments: " + orderedDepartments);
        System.out.println("Sorted departments: " + sortedDepartments);
        System.out.println("By ID: " + byId);
        System.out.println("Salary by name: " + salaryByName);
        System.out.println("First by department: " + firstEmployeeByDepartment);
        System.out.println("Latest by department: " + latestEmployeeByDepartment);
        System.out.println("Highest paid: " + highestPaidByDepartment);
        System.out.println("LinkedHashMap: " + linkedMap);
        System.out.println("TreeMap: " + treeMap);
        System.out.println("Quantity by product: " + quantityByProduct);
        System.out.println("Uppercase names: " + upperCaseNames);
        System.out.println("Concurrent map: " + concurrentMap);
        System.out.println("Stream.toList(): " + immutableNames);
    }
}
```

---

# 19. Interview Scenario — Duplicate Employee IDs 🔥🔥🔥

**Question:** What happens here?

```java
Map<Integer, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity()
        ));
```

### Answer

If duplicate IDs occur, the two-argument `toMap()` throws `IllegalStateException`.

Use a merge function when duplicates are valid:

```java
(existing, replacement) -> existing
```

or:

```java
(existing, replacement) -> replacement
```

or a business-specific merge such as:

```java
BinaryOperator.maxBy(
        Comparator.comparingInt(Employee::getSalary)
)
```

---

# 20. Interview Scenario — Keep Highest Salary on Duplicate Key ⭐⭐⭐⭐⭐

```java
Map<String, Employee> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getDepartment,
                Function.identity(),
                BinaryOperator.maxBy(
                        Comparator.comparingInt(Employee::getSalary)
                )
        ));
```

### 2-minute explanation

```text
Key       → department
Value     → employee
Duplicate → compare salaries
Winner    → employee with maximum salary
```

---

# 21. Interview Scenario — Product Quantity Aggregation

```java
Map<Integer, Integer> result = products.stream()
        .collect(Collectors.toMap(
                Product::getId,
                Product::getQuantity,
                Integer::sum
        ));
```

This is a common real-world aggregation pattern.

---

# 22. Interview Scenario — Remove Duplicate Departments

```java
Set<String> result = employees.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toSet());
```

If insertion order matters:

```java
Set<String> result = employees.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toCollection(LinkedHashSet::new));
```

---

# 23. Interview Scenario — Sorted Unique Values

```java
Set<String> result = employees.stream()
        .map(Employee::getDepartment)
        .collect(Collectors.toCollection(TreeSet::new));
```

Now the Set is sorted according to its natural ordering.

---

# 24. Interview Scenario — Map ID to Name

```java
Map<Integer, String> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Employee::getName
        ));
```

---

# 25. Interview Scenario — Map ID to DTO

```java
Map<Integer, EmployeeDto> result = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                e -> new EmployeeDto(e.getId(), e.getName())
        ));
```

The value mapper can create any required result type.

---

# 26. Interview Scenario — Convert List to Map 🔥

```java
List<Employee> employees = ...;

Map<Integer, Employee> employeeMap = employees.stream()
        .collect(Collectors.toMap(
                Employee::getId,
                Function.identity()
        ));
```

Then lookup becomes:

```java
Employee employee = employeeMap.get(101);
```

---

# 27. Interview Scenario — Convert Map to List

```java
List<Employee> employees = employeeMap.values()
        .stream()
        .collect(Collectors.toList());
```

Keys:

```java
List<Integer> ids = employeeMap.keySet()
        .stream()
        .collect(Collectors.toList());
```

Entries:

```java
List<Map.Entry<Integer, Employee>> entries = employeeMap.entrySet()
        .stream()
        .collect(Collectors.toList());
```

---

# 28. Interview Scenario — Convert List to Set

```java
Set<String> uniqueNames = names.stream()
        .collect(Collectors.toSet());
```

This removes duplicate values according to Set equality semantics.

---

# 29. `toMap()` Null Key / Null Value Trap 🔥

Do not blindly assume all Map implementations and collectors have identical null behavior.

For `Collectors.toMap()`, a null value from the value mapper can result in a `NullPointerException` during collection in standard JDK implementations.

Validate or normalize data first when nulls are possible:

```java
Map<Integer, String> result = employees.stream()
        .filter(e -> e.getName() != null)
        .collect(Collectors.toMap(
                Employee::getId,
                Employee::getName
        ));
```

---

# 30. `toMap()` vs `groupingBy()` — Decision Tree 🧠

```text
Need a Map?
   │
   ├── One value per key
   │       ↓
   │    toMap()
   │
   └── Multiple values per key
           ↓
       groupingBy()
```

If duplicate keys are possible with `toMap()`:

```text
Need business merge?
      ↓
add mergeFunction
```

---

# 31. 25 Interview Questions 🔥🔥🔥

1. What does `Collectors.toList()` do?
2. Difference between `Stream.toList()` and `Collectors.toList()`?
3. Is the List returned by `Stream.toList()` modifiable?
4. Does `Collectors.toList()` guarantee ArrayList?
5. What does `toSet()` do?
6. Does `toSet()` guarantee ordering?
7. How do you create a `LinkedHashSet` using a stream?
8. How do you create a `TreeSet` using a stream?
9. What does `toMap()` do?
10. What are the common `toMap()` overloads?
11. What happens on duplicate keys with two-argument `toMap()`?
12. Why is a merge function needed?
13. How do you keep the first duplicate value?
14. How do you keep the latest duplicate value?
15. How do you keep the maximum salary on duplicate keys?
16. How do you sum duplicate numeric values?
17. What is the map factory in `toMap()`?
18. How do you create a `LinkedHashMap` with `toMap()`?
19. How do you create a `TreeMap` with `toMap()`?
20. What is `toConcurrentMap()`?
21. Difference between `toMap()` and `groupingBy()`?
22. How do you convert List → Map?
23. How do you convert Map → List?
24. How do you remove duplicates from a stream?
25. Explain the duplicate-key problem in `toMap()` in 2 minutes.

---

# 32. Coding Challenges 🔥🔥🔥

### Challenge 1
Convert `List<Employee>` to `Map<Integer, Employee>`.

### Challenge 2
Convert employees to `Map<Integer, String>` containing ID → name.

### Challenge 3
Extract unique departments into a Set.

### Challenge 4
Extract sorted unique departments into a `TreeSet`.

### Challenge 5
Create insertion-order unique departments using `LinkedHashSet`.

### Challenge 6
Create name → salary map.

### Challenge 7
Handle duplicate employee names with a merge function.

### Challenge 8
Keep the employee with the highest salary when department is duplicated.

### Challenge 9
Keep the latest employee for duplicate department keys.

### Challenge 10
Sum quantities for duplicate product IDs.

### Challenge 11
Create an ordered `LinkedHashMap` from a stream.

### Challenge 12
Create a sorted `TreeMap` from a stream.

### Challenge 13
Convert a Map's values into a List.

### Challenge 14
Convert a Map's keys into a List.

### Challenge 15
Convert a Map's entries into a List.

### Challenge 16 ⭐⭐⭐⭐⭐
Convert employee data into:

```text
Department → Highest Paid Employee
```

using only `toMap()`.

### Challenge 17
Convert products into:

```text
Product ID → Total Quantity
```

### Challenge 18
Create:

```text
Employee ID → Uppercase Employee Name
```

### Challenge 19
Create a `TreeSet` of employee names sorted naturally.

### Challenge 20
Create a `LinkedHashSet` of employee names preserving first-seen order.

### Challenge 21 — 5-Year Interview Level 🔥
Given duplicate transactions, aggregate them into:

```text
Account ID → Total Transaction Amount
```

### Challenge 22
Given duplicate orders, keep the order with the highest amount per customer.

### Challenge 23
Create a `ConcurrentMap` from a parallel stream.

### Challenge 24
Explain why `toMap()` is not a substitute for `groupingBy()` when multiple values belong to one key.

### Challenge 25 — Production Scenario 🔥🔥🔥
Given API records with duplicate IDs, create a deterministic map using a business-defined merge rule and explain why the merge function is required.

---

# 33. Common Mistakes ❌

### ❌ Mistake 1 — Ignoring duplicate keys

```java
Collectors.toMap(Employee::getDepartment, Function.identity())
```

If department is not unique, this can throw `IllegalStateException`.

### ❌ Mistake 2 — Assuming `toSet()` is sorted

It is not a sorted Set by default.

### ❌ Mistake 3 — Assuming `toList()` means `ArrayList`

The collector does not promise a particular concrete implementation.

### ❌ Mistake 4 — Confusing `Stream.toList()` with `Collectors.toList()`

Their mutability guarantees differ.

### ❌ Mistake 5 — Using `toMap()` when one key naturally has many values

Use `groupingBy()` instead.

### ❌ Mistake 6 — Forgetting the merge function's business meaning

Do not blindly write:

```java
(a, b) -> a
```

Ask whether the correct business rule is first, latest, maximum, sum, concatenation, etc.

---

# 34. Final Revision Sheet 🧠

```text
toList()
────────────────────
Stream → List
```

```text
toSet()
────────────────────
Stream → Set
Duplicates removed
No ordering guarantee
```

```text
toCollection(...)
────────────────────
Stream → specific collection implementation
```

```text
toMap(key, value)
────────────────────
Stream → Map
Duplicate key → IllegalStateException
```

```text
toMap(key, value, merge)
────────────────────
Duplicate key → business merge rule
```

```text
toMap(key, value, merge, mapFactory)
────────────────────
Control Map implementation
```

```text
toConcurrentMap()
────────────────────
ConcurrentMap result
```

### Golden Rules

```text
Need List?                       → toList()
Need unique values?              → toSet()
Need specific collection type?   → toCollection()
Need one value per key?          → toMap()
Duplicate key possible?          → toMap() + merge function
Need many values per key?        → groupingBy()
Need sorted Map?                 → toMap(..., TreeMap::new)
Need insertion-order Map?        → toMap(..., LinkedHashMap::new)
Need concurrent Map?             → toConcurrentMap()
```

---

# 35. 2-Minute Interview Script 🎤

> “`Collectors.toList()`, `toSet()` and `toMap()` are common terminal collectors used to materialize stream results. I use `toList()` when I need a List and `toSet()` when duplicates should be removed. If I need a specific collection implementation such as `LinkedHashSet`, `TreeSet` or `ArrayList`, I use `toCollection()`. I use `toMap()` when the result naturally has one value per key. The important issue with `toMap()` is duplicate keys: the two-argument overload throws `IllegalStateException`, so when duplicates are valid I provide a merge function such as keeping the first value, keeping the latest value, summing values, or selecting the maximum salary. If one key can have multiple elements, I normally prefer `groupingBy()`. The map factory overload lets me choose implementations such as `LinkedHashMap` or `TreeMap`, and `toConcurrentMap()` is available when a concurrent result is required.”

---

# 🧪 Complete Practice Code

[GitHub — 9.16 `toMap()` / `toSet()` / `toList()` Deep Dive Practice Code](https://github.com/Nirbhay91/Core-Java/tree/master/09-Java-8-Modern-Features/16-Collectors-toMap-toSet-toList-Deep-Dive)

---

## Navigation

[← 9.15 — `groupingBy()` / `partitioningBy()` Deep Dive](../15-Collectors-groupingBy-partitioningBy-Deep-Dive/README.md)

**Current → 9.16 — `Collectors.toMap()` / `toSet()` / `toList()` Deep Dive → ✅ Completed**

**Next → 9.17 — `Collectors.joining()` / `mapping()` / `collectingAndThen()`**