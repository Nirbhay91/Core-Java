# 8.31 — `thenApply` / `thenCompose` / `thenCombine`

> **Goal:** Master the three most important `CompletableFuture` composition patterns: transform one result, flatten a dependent asynchronous operation, and combine independent asynchronous results.

---

## 1. The Core Mental Model ⭐⭐⭐⭐⭐

```text
thenApply
Future<T> → T transformation → Future<R>

thenCompose
Future<T> → async Future<R> → Future<R>

thenCombine
Future<T> + Future<U> → combine(T, U) → Future<R>
```

### Easy memory trick 🧠

```text
Apply   = change the value
Compose = chain another Future
Combine = join two independent Futures
```

---

# 2. `thenApply()` — Transform a Result ⭐⭐⭐⭐⭐

Use `thenApply()` when the next operation is a **synchronous transformation** of the previous result.

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "java")
                .thenApply(String::toUpperCase);
```

Conceptually:

```text
Future<String>
     ↓
thenApply
     ↓
Future<String>
```

It is similar to `map()` in functional programming.

---

# 3. Practice — Basic `thenApply()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ThenApplyDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> "java")
                        .thenApply(String::toUpperCase);

        System.out.println(future.join());
    }
}
```

Output:

```text
JAVA
```

---

# 4. Multiple `thenApply()` Stages ⭐⭐⭐⭐⭐

Stages can be chained:

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "java")
                .thenApply(String::toUpperCase)
                .thenApply(value -> value + " CONCURRENCY");
```

Pipeline:

```text
"java"
  ↓
"JAVA"
  ↓
"JAVA CONCURRENCY"
```

---

# 5. Practice — Transformation Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ThenApplyPipelineDemo {

    public static void main(String[] args) {
        CompletableFuture<Integer> future =
                CompletableFuture.supplyAsync(() -> 10)
                        .thenApply(value -> value * 2)
                        .thenApply(value -> value + 5);

        System.out.println(future.join());
    }
}
```

Result:

```text
25
```

---

# 6. `thenApply()` Does NOT Create a New Async Task ⭐⭐⭐⭐⭐

Important interview point:

```java
.thenApply(value -> transform(value))
```

is a non-`Async` continuation.

Its execution depends on completion timing and the thread completing the previous stage; it does **not** automatically mean "submit this stage to the common pool."

If you specifically want asynchronous execution, use:

```java
.thenApplyAsync(value -> transform(value))
```

and optionally provide an executor.

---

# 7. `thenApply()` vs `thenApplyAsync()` ⭐⭐⭐⭐⭐

| Method | Execution model |
|---|---|
| `thenApply()` | Synchronous/non-async continuation semantics |
| `thenApplyAsync()` | Asynchronous execution via default async facility |
| `thenApplyAsync(fn, executor)` | Asynchronous execution using supplied executor |

Do not say that `thenApply()` always runs on the same thread. Its execution thread depends on when/how the previous stage completes.

---

# 8. Practice — `thenApplyAsync()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ThenApplyAsyncDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    System.out.println("Supplier: " +
                            Thread.currentThread().getName());
                    return "java";
                }).thenApplyAsync(value -> {
                    System.out.println("Apply: " +
                            Thread.currentThread().getName());
                    return value.toUpperCase();
                });

        System.out.println(future.join());
    }
}
```

Run this and observe the thread names.

---

# 9. When to Use `thenApply()` ⭐⭐⭐⭐⭐

Use it for transformations such as:

```text
String → uppercase String
UserDTO → UserResponse
Order → OrderSummary
Integer → calculated Integer
JSON → DTO
```

The key property is:

> **The next operation returns a normal value `R`, not another `CompletableFuture<R>`.**

---

# 10. The Problem: Returning a `CompletableFuture` from `thenApply()` ⭐⭐⭐⭐⭐

Suppose:

```java
CompletableFuture<User> userFuture = loadUser();
```

and:

```java
CompletableFuture<Order> loadOrders(User user)
```

If we write:

```java
CompletableFuture<CompletableFuture<Order>> result =
        userFuture.thenApply(user -> loadOrders(user));
```

we get a **nested future**.

```text
Future<User>
    ↓ thenApply
Future<Future<Order>>
```

Usually this is not what we want.

---

# 11. `thenCompose()` — Flatten Dependent Async Work ⭐⭐⭐⭐⭐

Use `thenCompose()` when the next operation itself returns a `CompletionStage` / `CompletableFuture`.

```java
CompletableFuture<Order> orders =
        userFuture.thenCompose(user -> loadOrders(user));
```

Conceptually:

```text
Future<User>
     ↓
async loadOrders(User)
     ↓
Future<Order>
```

This is similar to `flatMap()` in functional programming.

---

# 12. Practice — Basic `thenCompose()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ThenComposeDemo {

    static CompletableFuture<String> loadUser() {
        return CompletableFuture.supplyAsync(() -> "Nirbhay");
    }

    static CompletableFuture<String> loadOrders(String user) {
        return CompletableFuture.supplyAsync(() ->
                user + " -> Orders loaded");
    }

    public static void main(String[] args) {
        CompletableFuture<String> result =
                loadUser()
                        .thenCompose(ThenComposeDemo::loadOrders);

        System.out.println(result.join());
    }
}
```

---

# 13. `thenApply()` vs `thenCompose()` ⭐⭐⭐⭐⭐

### `thenApply`

```java
CompletableFuture<String> result =
        future.thenApply(value -> value.toUpperCase());
```

Returns:

```text
Future<String>
```

### `thenCompose`

```java
CompletableFuture<String> result =
        future.thenCompose(value -> asyncOperation(value));
```

where:

```java
CompletableFuture<String> asyncOperation(String value)
```

Returns one flattened:

```text
Future<String>
```

### If you use `thenApply()` with an async-returning function:

```text
Future<Future<String>>
```

### If you use `thenCompose()`:

```text
Future<String>
```

---

# 14. Practice — See the Difference ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ApplyVsComposeDemo {

    static CompletableFuture<String> asyncUpper(String value) {
        return CompletableFuture.supplyAsync(value::toUpperCase);
    }

    public static void main(String[] args) {
        CompletableFuture<CompletableFuture<String>> nested =
                CompletableFuture.completedFuture("java")
                        .thenApply(ApplyVsComposeDemo::asyncUpper);

        CompletableFuture<String> flat =
                CompletableFuture.completedFuture("java")
                        .thenCompose(ApplyVsComposeDemo::asyncUpper);

        System.out.println("Nested = " + nested.join().join());
        System.out.println("Flat = " + flat.join());
    }
}
```

---

# 15. Dependent Async Calls ⭐⭐⭐⭐⭐

Use `thenCompose()` when call B depends on the result of call A:

```text
loadCustomer()
      ↓ customerId
loadOrders(customerId)
      ↓ orders
loadPaymentStatus(orders)
```

Each next call depends on the previous result.

This is a classic `thenCompose()` use case.

---

# 16. Practice — Three Dependent Calls ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class DependentCallsDemo {

    static CompletableFuture<String> customer() {
        return CompletableFuture.completedFuture("C101");
    }

    static CompletableFuture<String> orders(String customerId) {
        return CompletableFuture.completedFuture(
                "Orders for " + customerId);
    }

    static CompletableFuture<String> payment(String orders) {
        return CompletableFuture.completedFuture(
                "Payment status for [" + orders + "] = SUCCESS");
    }

    public static void main(String[] args) {
        CompletableFuture<String> result = customer()
                .thenCompose(DependentCallsDemo::orders)
                .thenCompose(DependentCallsDemo::payment);

        System.out.println(result.join());
    }
}
```

---

# 17. `thenCombine()` — Combine Independent Futures ⭐⭐⭐⭐⭐

Use `thenCombine()` when you have **two independent asynchronous computations** and need both results to produce one final result.

```java
CompletableFuture<String> a = loadA();
CompletableFuture<String> b = loadB();

CompletableFuture<String> result =
        a.thenCombine(b, (x, y) -> x + y);
```

Mental model:

```text
Future<A> ─────┐
               ├──→ combine(A, B) → Future<R>
Future<B> ─────┘
```

---

# 18. Practice — Basic `thenCombine()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ThenCombineDemo {

    public static void main(String[] args) {
        CompletableFuture<String> user =
                CompletableFuture.supplyAsync(() -> "Nirbhay");

        CompletableFuture<String> role =
                CompletableFuture.supplyAsync(() -> "Java Developer");

        CompletableFuture<String> result =
                user.thenCombine(role,
                        (u, r) -> u + " - " + r);

        System.out.println(result.join());
    }
}
```

---

# 19. `thenCombine()` Means Independent Work ⭐⭐⭐⭐⭐

Example:

```text
User Service ──────┐
                   ├──→ API response
Recommendation ────┘
```

Neither operation needs the other's result to start.

Therefore:

```java
thenCombine()
```

is appropriate.

---

# 20. `thenCompose()` vs `thenCombine()` ⭐⭐⭐⭐⭐

| Question | Use |
|---|---|
| Does next async call depend on previous result? | `thenCompose()` |
| Are two async calls independent? | `thenCombine()` |
| Do I only transform one result? | `thenApply()` |

### Memory trick

```text
A → transform B             = thenApply
A → async B                 = thenCompose
A ─┐
   ├→ C                     = thenCombine
B ─┘
```

---

# 21. Practice — Realistic Service Scenario ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ServiceCompositionDemo {

    static CompletableFuture<String> loadCustomer() {
        return CompletableFuture.supplyAsync(() -> "Customer-101");
    }

    static CompletableFuture<String> loadOrders(String customer) {
        return CompletableFuture.supplyAsync(() ->
                "Orders(" + customer + ")");
    }

    static CompletableFuture<String> loadRecommendations() {
        return CompletableFuture.supplyAsync(() ->
                "Recommendations");
    }

    public static void main(String[] args) {
        CompletableFuture<String> customerOrders =
                loadCustomer()
                        .thenCompose(ServiceCompositionDemo::loadOrders);

        CompletableFuture<String> response =
                customerOrders.thenCombine(
                        loadRecommendations(),
                        (orders, recommendations) ->
                                orders + " + " + recommendations);

        System.out.println(response.join());
    }
}
```

Pipeline:

```text
loadCustomer()
      ↓
thenCompose(loadOrders)
      ↓
customerOrders
      ↓
thenCombine(loadRecommendations)
      ↓
response
```

---

# 22. `thenCombine()` vs `thenCompose()` Interview Scenario 🏆

### Question

> User details and recommendations are independent calls. Which method?

**Answer:** `thenCombine()`.

### Question

> After getting the user ID, you must call the order service using that ID. Which method?

**Answer:** `thenCompose()`.

### Question

> Convert a `User` object into a `UserDTO` without making another async call. Which method?

**Answer:** `thenApply()`.

---

# 23. `thenAccept()` and `thenRun()` Context ⭐⭐⭐⭐

Although this topic focuses on three methods, understand the neighboring methods:

### `thenAccept()`

Consumes a result but does not return a new value:

```java
future.thenAccept(System.out::println);
```

Returns:

```text
CompletableFuture<Void>
```

### `thenRun()`

Runs an action after completion without receiving the previous result:

```java
future.thenRun(() -> System.out.println("Done"));
```

Also returns:

```text
CompletableFuture<Void>
```

---

# 24. Practice — Apply / Accept / Run ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ApplyAcceptRunDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.completedFuture("java")
                        .thenApply(String::toUpperCase);

        future.thenAccept(value ->
                System.out.println("Value = " + value));

        future.thenRun(() ->
                System.out.println("Pipeline completed"));

        future.join();
    }
}
```

---

# 25. `thenApply()` Is for Mapping ⭐⭐⭐⭐⭐

Think:

```text
T → R
```

Example:

```java
User → UserDTO
```

```java
.thenApply(user -> toDto(user))
```

---

# 26. `thenCompose()` Is for Flattening ⭐⭐⭐⭐⭐

Think:

```text
T → Future<R>
```

Example:

```java
User → Future<List<Order>>
```

```java
.thenCompose(user -> loadOrders(user))
```

It avoids:

```text
Future<Future<R>>
```

---

# 27. `thenCombine()` Is for Joining ⭐⭐⭐⭐⭐

Think:

```text
Future<A> + Future<B> → R
```

Example:

```java
userFuture.thenCombine(
        orderFuture,
        (user, orders) -> buildResponse(user, orders));
```

---

# 28. Common Mistake — Using `thenCompose()` for Independent Calls ⭐⭐⭐⭐⭐

Incorrect conceptual design:

```java
loadUser()
    .thenCompose(user -> loadRecommendations());
```

If recommendations do not depend on `user`, this unnecessarily models the second operation as dependent on the first.

Better:

```java
CompletableFuture<User> user = loadUser();
CompletableFuture<Recommendations> recommendations =
        loadRecommendations();

user.thenCombine(recommendations,
        (u, r) -> buildResponse(u, r));
```

---

# 29. Common Mistake — Using `thenApply()` for an Async Method ⭐⭐⭐⭐⭐

If:

```java
CompletableFuture<Order> loadOrders(User user)
```

then:

```java
userFuture.thenApply(user -> loadOrders(user));
```

creates:

```text
CompletableFuture<CompletableFuture<Order>>
```

Usually use:

```java
userFuture.thenCompose(user -> loadOrders(user));
```

---

# 30. Common Mistake — Assuming `thenApply()` Always Runs on Previous Thread ⭐⭐⭐⭐⭐

Do not give this oversimplified interview answer:

> "thenApply always executes on the same thread."

More accurate:

> `thenApply()` is a non-async continuation. Its execution may occur in the thread that completes the previous stage, depending on completion timing and implementation behavior. `thenApplyAsync()` requests asynchronous execution through an executor.

---

# 31. Custom Executor with `thenApplyAsync()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CustomExecutorCompositionDemo {

    public static void main(String[] args) {
        ExecutorService executor =
                Executors.newFixedThreadPool(2);

        try {
            CompletableFuture<String> future =
                    CompletableFuture.supplyAsync(
                            () -> "java", executor)
                    .thenApplyAsync(
                            String::toUpperCase, executor)
                    .thenApplyAsync(
                            value -> value + " CONCURRENCY", executor);

            System.out.println(future.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 32. Exception Propagation Through Composition ⭐⭐⭐⭐⭐

If an earlier stage completes exceptionally, normal transformation stages are not executed as if a normal value existed.

Example:

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Failure");
        }).thenApply(String::toUpperCase);
```

The resulting future remains exceptional.

Detailed recovery methods are covered in **8.32 — Exception Handling in `CompletableFuture`**.

---

# 33. Practice — Failure Stops Normal Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class PipelineFailureDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Service failed");
                }).thenApply(value -> {
                    System.out.println("This should not run normally");
                    return value.toUpperCase();
                });

        try {
            future.join();
        } catch (Exception e) {
            System.out.println("Pipeline failed: " + e.getCause());
        }
    }
}
```

---

# 34. Sequential vs Independent Composition ⭐⭐⭐⭐⭐

### Dependent

```text
A → B → C
```

Use:

```java
thenCompose()
```

### Independent

```text
A ─┐
   ├→ C
B ─┘
```

Use:

```java
thenCombine()
```

### Transformation

```text
A → transformed A
```

Use:

```java
thenApply()
```

---

# 35. Latency Thinking ⭐⭐⭐⭐⭐

Suppose two independent remote calls each take about 500 ms.

Sequential:

```text
A: 500ms
B: 500ms
Total ≈ 1000ms
```

Independent concurrent execution:

```text
A: ───────── 500ms
B: ───────── 500ms
Total ≈ 500ms + overhead
```

This is only a conceptual model. Real latency depends on executor capacity, network behavior, downstream services, queueing, and resource contention.

---

# 36. Practice — Independent Calls ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class IndependentCallsDemo {

    static CompletableFuture<String> serviceA() {
        return CompletableFuture.supplyAsync(() -> {
            sleep(300);
            return "A";
        });
    }

    static CompletableFuture<String> serviceB() {
        return CompletableFuture.supplyAsync(() -> {
            sleep(300);
            return "B";
        });
    }

    static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        CompletableFuture<String> a = serviceA();
        CompletableFuture<String> b = serviceB();

        String result = a.thenCombine(b,
                (x, y) -> x + y).join();

        long elapsed = System.currentTimeMillis() - start;

        System.out.println("Result = " + result);
        System.out.println("Elapsed ≈ " + elapsed + " ms");
    }
}
```

Do not expect exact timing because scheduling and machine load vary.

---

# 37. Advanced Interview Scenario 🏆

### Requirement

```text
1. Get customer
2. Using customer ID, get orders
3. Independently get recommendations
4. Combine orders + recommendations
5. Build response
```

Correct structure:

```text
             loadCustomer()
                  ↓
             thenCompose
                  ↓
             loadOrders()
                  ↓
             ordersFuture
                  │
                  ├───────┐
                  │       │
                  │   recommendationsFuture
                  │       │
                  └─ thenCombine ─┘
                          ↓
                     final response
```

This is a realistic senior-level composition pattern.

---

# 38. Practice — Complete Interview Scenario ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class InterviewCompositionDemo {

    static CompletableFuture<String> getCustomer() {
        return CompletableFuture.supplyAsync(() -> "Customer-101");
    }

    static CompletableFuture<String> getOrders(String customer) {
        return CompletableFuture.supplyAsync(() ->
                "Orders for " + customer);
    }

    static CompletableFuture<String> getRecommendations() {
        return CompletableFuture.supplyAsync(() ->
                "Recommendations loaded");
    }

    public static void main(String[] args) {
        CompletableFuture<String> ordersFuture =
                getCustomer()
                        .thenCompose(InterviewCompositionDemo::getOrders);

        CompletableFuture<String> recommendationFuture =
                getRecommendations();

        CompletableFuture<String> response =
                ordersFuture.thenCombine(
                        recommendationFuture,
                        (orders, recommendations) ->
                                "Response {" + orders +
                                ", " + recommendations + "}");

        System.out.println(response.join());
    }
}
```

---

# 39. Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is `thenApply()`?

> It transforms the successful result of a stage into another value and returns a new `CompletionStage`.

### Q2. What is `thenCompose()`?

> It chains a dependent asynchronous operation and flattens the resulting stage, avoiding `CompletableFuture<CompletableFuture<T>>`.

### Q3. What is `thenCombine()`?

> It combines two independently completing stages once both are complete and produces a new result from both values.

### Q4. `thenApply()` vs `thenCompose()`?

> `thenApply()` is for `T → R`; `thenCompose()` is for `T → CompletionStage<R>`.

### Q5. `thenCompose()` vs `thenCombine()`?

> `thenCompose()` models dependency between stages; `thenCombine()` models independent stages whose results must be combined.

### Q6. Does `thenApply()` always execute on the same thread?

> No. It is a non-async continuation, and execution depends on completion timing and the thread completing the previous stage. `thenApplyAsync()` explicitly requests asynchronous execution.

### Q7. Why does `thenApply()` sometimes create nested futures?

> Because if the function itself returns a `CompletableFuture`, `thenApply()` maps it as an ordinary value. `thenCompose()` flattens that nested stage.

---

# 40. Common Interview Traps 🚨

```text
❌ thenApply = async API call

❌ thenCompose = combine two independent futures

❌ thenCombine = dependent chain

❌ thenApply always runs on the same thread

❌ thenApplyAsync guarantees a brand-new thread

❌ thenCompose is just another name for thenApply
```

Correct:

```text
thenApply   → synchronous transformation
thenCompose → dependent async composition / flattening
thenCombine → independent async results → one result
```

---

# 41. 2-Minute Interview Answer 🏆

> **"In `CompletableFuture`, I use `thenApply()` when I only need to transform the result of the current stage, for example converting a `User` into a `UserDTO`. I use `thenCompose()` when the next operation is asynchronous and depends on the previous result, such as getting orders after obtaining a customer ID. It flattens the nested future that `thenApply()` would otherwise create. I use `thenCombine()` when two asynchronous operations are independent and I need both results to create one final result, such as combining customer details with recommendations. The key distinction is transformation versus dependent async composition versus independent combination."**

---

# 42. Quick Revision ⭐⭐⭐⭐⭐

```text
                    CompletableFuture
                           │
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
     thenApply       thenCompose       thenCombine
          │                │                │
       T → R          T → Future<R>     Future<A> + Future<B>
          │                │                │
       mapping          flattening       combining
          │                │                │
     User → DTO       User → Orders      User + Orders → Response
```

### Golden Rule 🧠

> **Apply changes a value. Compose chains a dependent future. Combine joins independent futures.**

---

# 43. 💻 Practice Checklist

- [ ] Basic `thenApply()`
- [ ] Multiple `thenApply()` stages
- [ ] `thenApply()` vs `thenApplyAsync()`
- [ ] Observe continuation thread names
- [ ] Identify synchronous transformations
- [ ] Basic `thenCompose()`
- [ ] Understand nested `CompletableFuture`
- [ ] Flatten nested futures with `thenCompose()`
- [ ] Build dependent async service calls
- [ ] Basic `thenCombine()`
- [ ] Combine independent service calls
- [ ] Compare `thenApply` vs `thenCompose`
- [ ] Compare `thenCompose` vs `thenCombine`
- [ ] Practice `thenAccept()` / `thenRun()` context
- [ ] Use custom executor with async composition
- [ ] Understand exception propagation
- [ ] Model sequential vs independent latency
- [ ] Solve the customer → orders → recommendations scenario
- [ ] Answer all interview questions aloud
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.30 — `CompletableFuture` Fundamentals](../30-CompletableFuture-Fundamentals/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.32 — Exception Handling in `CompletableFuture`**