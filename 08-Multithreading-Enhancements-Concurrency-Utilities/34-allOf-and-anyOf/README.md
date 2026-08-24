# 8.34 — `allOf()` / `anyOf()`

> **Goal:** Learn how `CompletableFuture.allOf()` and `CompletableFuture.anyOf()` coordinate multiple asynchronous tasks, how to collect results correctly, handle failures, apply timeouts, and use these patterns in interview-level and production scenarios.

---

# 1. Core Mental Model ⭐⭐⭐⭐⭐

```text
allOf()
→ wait for ALL futures to complete
→ result type is CompletableFuture<Void>

anyOf()
→ complete when ANY one future completes
→ result type is CompletableFuture<Object>
```

Memory trick:

```text
ALL → sab complete hone ka wait
ANY → koi ek complete hote hi result
```

---

# 2. Why Do We Need `allOf()`?

Suppose we need data from three independent services:

```text
User Service        ─┐
Order Service       ─┼──→ wait for all → build response
Recommendation      ─┘
```

Sequential approach:

```java
getUser();
getOrders();
getRecommendations();
```

If each takes approximately 1 second, total latency can approach:

```text
1s + 1s + 1s = 3s
```

If they are independent, we can start them concurrently:

```text
User             ───── 1s ───┐
Orders           ───── 1s ───┼→ allOf
Recommendations  ───── 1s ───┘

Total ≈ 1s + coordination overhead
```

The actual latency depends on the executor, workload and system conditions.

---

# 3. Basic `allOf()` Syntax ⭐⭐⭐⭐⭐

```java
CompletableFuture<Void> all =
        CompletableFuture.allOf(future1, future2, future3);
```

Important:

```java
allOf()
```

does **not** return a collection of results.

It returns:

```java
CompletableFuture<Void>
```

This is one of the most common interview traps.

---

# 4. Practice — Basic `allOf()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class AllOfBasicDemo {

    public static void main(String[] args) {

        CompletableFuture<String> user =
                CompletableFuture.supplyAsync(() -> "User-101");

        CompletableFuture<String> orders =
                CompletableFuture.supplyAsync(() -> "5 Orders");

        CompletableFuture<String> recommendations =
                CompletableFuture.supplyAsync(() -> "3 Recommendations");

        CompletableFuture<Void> all =
                CompletableFuture.allOf(user, orders, recommendations);

        all.join();

        System.out.println("All tasks completed");
    }
}
```

### What to remember

```text
allOf()
   ↓
wait for all
   ↓
returns CompletableFuture<Void>
```

---

# 5. The Most Important Interview Question ⭐⭐⭐⭐⭐

### Q: How do you get results from `allOf()`?

You retrieve the result from the **original futures** after `allOf().join()` completes.

```java
all.join();

String userResult = user.join();
String orderResult = orders.join();
String recommendationResult = recommendations.join();
```

This is the pattern to memorize.

---

# 6. Complete Interview Code — `allOf()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class AllOfCompleteDemo {

    static CompletableFuture<String> getUser() {
        return CompletableFuture.supplyAsync(() -> "User-101");
    }

    static CompletableFuture<String> getOrders() {
        return CompletableFuture.supplyAsync(() -> "5 Orders");
    }

    static CompletableFuture<String> getRecommendations() {
        return CompletableFuture.supplyAsync(() -> "3 Recommendations");
    }

    public static void main(String[] args) {

        CompletableFuture<String> userFuture = getUser();
        CompletableFuture<String> orderFuture = getOrders();
        CompletableFuture<String> recommendationFuture =
                getRecommendations();

        CompletableFuture<Void> allFuture =
                CompletableFuture.allOf(
                        userFuture,
                        orderFuture,
                        recommendationFuture
                );

        allFuture.join();

        String user = userFuture.join();
        String orders = orderFuture.join();
        String recommendations = recommendationFuture.join();

        System.out.println(user);
        System.out.println(orders);
        System.out.println(recommendations);
    }
}
```

### Interview line

> **`allOf()` waits for all supplied futures, but it does not aggregate their results. I wait on `allOf()` and then retrieve each original future's result.**

---

# 7. Why Is `allOf()` Result `Void`? ⭐⭐⭐⭐⭐

Because Java does not know a common result type for an arbitrary number of futures:

```text
Future<String>
Future<Integer>
Future<Order>
Future<User>
```

Therefore:

```java
CompletableFuture.allOf(...)
```

only represents completion of the group.

Think:

```text
allOf = coordination
original futures = data
```

---

# 8. Collecting Results Into a List ⭐⭐⭐⭐⭐

A common interview pattern is:

```java
CompletableFuture<Void> all =
        CompletableFuture.allOf(futuresArray);

List<String> results = all.thenApply(v ->
        futures.stream()
                .map(CompletableFuture::join)
                .toList()
).join();
```

For Java versions where `Stream.toList()` is unavailable, use:

```java
.collect(Collectors.toList())
```

---

# 9. Complete Practice — `allOf()` + List Results ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

public class AllOfListDemo {

    public static void main(String[] args) {

        List<CompletableFuture<String>> futures = List.of(
                CompletableFuture.supplyAsync(() -> "Task-1"),
                CompletableFuture.supplyAsync(() -> "Task-2"),
                CompletableFuture.supplyAsync(() -> "Task-3")
        );

        CompletableFuture<Void> all =
                CompletableFuture.allOf(
                        futures.toArray(new CompletableFuture[0])
                );

        List<String> results = all.thenApply(v ->
                futures.stream()
                        .map(CompletableFuture::join)
                        .collect(Collectors.toList())
        ).join();

        System.out.println(results);
    }
}
```

Output:

```text
[Task-1, Task-2, Task-3]
```

The list order follows the order of the original `futures` list, not necessarily completion order.

---

# 10. `anyOf()` Fundamentals ⭐⭐⭐⭐⭐

`anyOf()` completes when the first supplied future completes.

```java
CompletableFuture<Object> first =
        CompletableFuture.anyOf(future1, future2, future3);
```

Important:

```java
anyOf()
→ CompletableFuture<Object>
```

because supplied futures can have different result types.

---

# 11. Practice — Basic `anyOf()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class AnyOfBasicDemo {

    public static void main(String[] args) {

        CompletableFuture<String> serviceA =
                CompletableFuture.supplyAsync(() -> {
                    sleep(1000);
                    return "Response-A";
                });

        CompletableFuture<String> serviceB =
                CompletableFuture.supplyAsync(() -> {
                    sleep(500);
                    return "Response-B";
                });

        CompletableFuture<String> serviceC =
                CompletableFuture.supplyAsync(() -> {
                    sleep(1500);
                    return "Response-C";
                });

        Object result = CompletableFuture
                .anyOf(serviceA, serviceB, serviceC)
                .join();

        System.out.println(result);
    }

    static void sleep(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

Expected result:

```text
Response-B
```

assuming no scheduling or runtime variation changes which future completes first.

---

# 12. `anyOf()` Does NOT Cancel the Losers Automatically ⭐⭐⭐⭐⭐

This is a major interview point.

If:

```text
A = 1000 ms
B = 500 ms
C = 1500 ms
```

and B wins:

```text
anyOf → B result
```

But A and C may continue running.

```text
B → winner → result returned
A → may still run
C → may still run
```

`anyOf()` is a completion race, not automatic cancellation of the other tasks.

---

# 13. Complete Interview Code — `anyOf()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class AnyOfCompleteDemo {

    static CompletableFuture<String> callService(
            String service,
            long delay) {

        return CompletableFuture.supplyAsync(() -> {
            sleep(delay);
            System.out.println(service + " completed");
            return service + " response";
        });
    }

    public static void main(String[] args) {

        CompletableFuture<String> serviceA =
                callService("Service-A", 1000);

        CompletableFuture<String> serviceB =
                callService("Service-B", 500);

        CompletableFuture<String> serviceC =
                callService("Service-C", 1500);

        CompletableFuture<Object> first =
                CompletableFuture.anyOf(
                        serviceA,
                        serviceB,
                        serviceC
                );

        Object result = first.join();

        System.out.println("Winner = " + result);

        // Important:
        // Service-A and Service-C may still continue running.
    }

    static void sleep(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

---

# 14. `allOf()` vs `anyOf()` ⭐⭐⭐⭐⭐

| Feature | `allOf()` | `anyOf()` |
|---|---|---|
| Waits for | All futures | First completion |
| Result | `CompletableFuture<Void>` | `CompletableFuture<Object>` |
| Main use | Parallel aggregation | Race/fallback |
| Other tasks | All must complete for group completion | Others may continue |
| Automatic cancellation | No | No |
| Result aggregation | Manual | Single winner result |

Memory:

```text
ALL → everyone
ANY → first one
```

---

# 15. Failure Behavior — `allOf()` ⭐⭐⭐⭐⭐

Suppose:

```text
A → success
B → failure
C → success
```

`allOf()` does not become normally successful merely because two succeeded.

The combined future completes exceptionally if one of the supplied futures completes exceptionally.

But the other futures may still complete independently.

---

# 16. Practice — `allOf()` With Failure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class AllOfFailureDemo {

    public static void main(String[] args) {

        CompletableFuture<String> success1 =
                CompletableFuture.completedFuture("A");

        CompletableFuture<String> failure =
                CompletableFuture.failedFuture(
                        new RuntimeException("B failed"));

        CompletableFuture<String> success2 =
                CompletableFuture.completedFuture("C");

        CompletableFuture<Void> all =
                CompletableFuture.allOf(
                        success1,
                        failure,
                        success2
                );

        try {
            all.join();
        } catch (Exception e) {
            System.out.println("Combined future failed: " + e);
        }

        System.out.println("A = " + success1.join());
        System.out.println("C = " + success2.join());
    }
}
```

---

# 17. Failure Behavior — `anyOf()` ⭐⭐⭐⭐⭐

`anyOf()` completes based on the **first completion**, not the first successful result.

This is a critical distinction.

Suppose:

```text
A → fails at 100 ms
B → succeeds at 500 ms
C → succeeds at 700 ms
```

Then:

```text
anyOf()
   ↓
A fails first
   ↓
combined future fails
```

It does **not** automatically wait for the first successful result.

---

# 18. Practice — First Completion Is Failure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class AnyOfFailureDemo {

    public static void main(String[] args) {

        CompletableFuture<String> failing =
                CompletableFuture.supplyAsync(() -> {
                    sleep(100);
                    throw new RuntimeException("Fast service failed");
                });

        CompletableFuture<String> slowSuccess =
                CompletableFuture.supplyAsync(() -> {
                    sleep(500);
                    return "Slow success";
                });

        try {
            Object result = CompletableFuture.anyOf(
                    failing,
                    slowSuccess
            ).join();

            System.out.println(result);
        } catch (Exception e) {
            System.out.println("anyOf failed: " + e);
        }
    }

    static void sleep(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

Interview line:

> **`anyOf()` means first completion, not first successful completion.**

---

# 19. How to Implement "First Successful Result" ⭐⭐⭐⭐⭐

This is different from `anyOf()`.

Requirement:

```text
Service A fails
Service B fails
Service C succeeds

→ return C
```

`anyOf()` alone is not enough because an early failure can win the race.

One simple approach is to convert failures into a value that indicates failure and then apply a success-selection strategy.

For production systems, the exact design should account for cancellation, timeouts, retries and resource cleanup.

---

# 20. Practice — First Successful Response Pattern ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

public class FirstSuccessfulDemo {

    public static void main(String[] args) {

        List<CompletableFuture<String>> futures = List.of(
                service("A", 200, false),
                service("B", 400, false),
                service("C", 600, true)
        );

        CompletableFuture<List<String>> safeResults =
                CompletableFuture.allOf(
                        futures.toArray(new CompletableFuture[0])
                ).thenApply(v -> futures.stream()
                        .map(CompletableFuture::join)
                        .collect(Collectors.toList()));

        System.out.println(safeResults.join());
    }

    static CompletableFuture<String> service(
            String name,
            long delay,
            boolean success) {

        return CompletableFuture.supplyAsync(() -> {
            sleep(delay);
            if (!success) {
                throw new RuntimeException(name + " failed");
            }
            return name + " success";
        }).exceptionally(ex -> "FAILED: " + name);
    }

    static void sleep(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

This example demonstrates a **collect-and-select** pattern. It is not the same as a true latency-race that returns immediately on the first success.

---

# 21. `allOf()` + `thenApply()` ⭐⭐⭐⭐⭐

A powerful pattern:

```java
CompletableFuture<List<String>> result =
        CompletableFuture.allOf(futures)
                .thenApply(v -> futures.stream()
                        .map(CompletableFuture::join)
                        .collect(Collectors.toList()));
```

Mental model:

```text
start all
   ↓
allOf
   ↓
all complete
   ↓
join each original future
   ↓
collect results
```

---

# 22. Custom Executor With `allOf()` ⭐⭐⭐⭐⭐

`allOf()` itself coordinates futures; the executor is normally selected when creating the individual asynchronous tasks.

```java
CompletableFuture.supplyAsync(task, executor)
```

Then:

```java
CompletableFuture.allOf(f1, f2, f3)
```

The executor controls the work performed by the individual async stages, not `allOf()` magically moving all work to one executor.

---

# 23. Complete Interview Code — `allOf()` + Custom Executor ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.Collectors;

public class AllOfCustomExecutorDemo {

    public static void main(String[] args) {

        ExecutorService executor =
                Executors.newFixedThreadPool(3);

        try {
            List<CompletableFuture<String>> futures = List.of(
                    CompletableFuture.supplyAsync(
                            () -> "User", executor),
                    CompletableFuture.supplyAsync(
                            () -> "Orders", executor),
                    CompletableFuture.supplyAsync(
                            () -> "Recommendations", executor)
            );

            CompletableFuture<List<String>> result =
                    CompletableFuture.allOf(
                            futures.toArray(new CompletableFuture[0])
                    ).thenApply(v -> futures.stream()
                            .map(CompletableFuture::join)
                            .collect(Collectors.toList()));

            System.out.println(result.join());
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 24. Timeout + `allOf()` ⭐⭐⭐⭐⭐

In production, waiting forever is usually undesirable.

You can apply a timeout to the combined future:

```java
CompletableFuture<Void> all =
        CompletableFuture.allOf(f1, f2, f3)
                .orTimeout(2, TimeUnit.SECONDS);
```

Remember:

```text
Timeout of combined future
≠ automatically guaranteed cancellation of every underlying task
```

Design cancellation deliberately when resources matter.

---

# 25. Practice — `allOf()` + Timeout ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class AllOfTimeoutDemo {

    public static void main(String[] args) {

        CompletableFuture<String> a =
                CompletableFuture.supplyAsync(() -> {
                    sleep(1000);
                    return "A";
                });

        CompletableFuture<String> b =
                CompletableFuture.supplyAsync(() -> {
                    sleep(3000);
                    return "B";
                });

        CompletableFuture<Void> all =
                CompletableFuture.allOf(a, b)
                        .orTimeout(2, TimeUnit.SECONDS);

        try {
            all.join();
            System.out.println("All completed");
        } catch (Exception e) {
            System.out.println("Combined timeout/failure: " + e);
        }
    }

    static void sleep(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

---

# 26. `anyOf()` as a Race Between Replicas ⭐⭐⭐⭐⭐

Real-world scenario:

```text
Primary service ─────── 800 ms
Replica service ─────── 400 ms
Cache service ───────── 200 ms

anyOf()
   ↓
fastest completion wins
```

Useful for:

```text
redundant providers
replicas
fallback endpoints
latency racing
```

But consider:

```text
cost
duplicate requests
side effects
cancellation
rate limits
```

before using request racing in production.

---

# 27. Practice — Replica Race ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class ReplicaRaceDemo {

    public static void main(String[] args) {

        CompletableFuture<String> primary =
                service("PRIMARY", 800);

        CompletableFuture<String> replica =
                service("REPLICA", 400);

        Object fastest = CompletableFuture
                .anyOf(primary, replica)
                .join();

        System.out.println("Fastest response = " + fastest);
    }

    static CompletableFuture<String> service(
            String name, long delay) {

        return CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(delay);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
            return name + " response";
        });
    }
}
```

---

# 28. Common Interview Trap — `join()` Inside `allOf()` ⭐⭐⭐⭐⭐

This is correct:

```java
CompletableFuture<Void> all =
        CompletableFuture.allOf(f1, f2, f3);

all.join();

String r1 = f1.join();
String r2 = f2.join();
String r3 = f3.join();
```

Why is the second `join()` not normally blocking?

Because after:

```java
all.join();
```

all supplied futures have completed.

The individual `join()` calls retrieve their already-completed results or propagate their exceptional completion.

---

# 29. Common Interview Trap — `anyOf()` Type ⭐⭐⭐⭐⭐

This:

```java
CompletableFuture.anyOf(f1, f2, f3)
```

returns:

```java
CompletableFuture<Object>
```

Therefore:

```java
Object result = CompletableFuture
        .anyOf(f1, f2, f3)
        .join();
```

If all futures return the same type, you may cast carefully:

```java
String result = (String) CompletableFuture
        .anyOf(f1, f2, f3)
        .join();
```

But remember that the API itself exposes `Object` because the inputs are not required to have the same result type.

---

# 30. `allOf()` vs Sequential Composition ⭐⭐⭐⭐⭐

### Sequential

```java
String a = futureA.join();
String b = futureB.join();
String c = futureC.join();
```

If the futures were already started independently, the operations may already be running concurrently, but your retrieval flow is sequential.

### Parallel coordination

```java
CompletableFuture.allOf(futureA, futureB, futureC)
        .join();
```

This explicitly communicates:

> These futures form one group whose completion I need to coordinate.

---

# 31. Production Scenario — Dashboard API 🏆

Suppose an endpoint needs:

```text
Profile
Orders
Payments
Recommendations
```

Independent calls:

```java
CompletableFuture<Profile> profile = getProfile();
CompletableFuture<List<Order>> orders = getOrders();
CompletableFuture<List<Payment>> payments = getPayments();
CompletableFuture<List<Product>> recommendations =
        getRecommendations();
```

Then:

```java
CompletableFuture.allOf(
        profile,
        orders,
        payments,
        recommendations
).thenApply(v ->
        buildDashboard(
                profile.join(),
                orders.join(),
                payments.join(),
                recommendations.join()
        )
);
```

This is one of the most useful interview examples.

---

# 32. Complete Production-Style Interview Example ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class DashboardService {

    public CompletableFuture<Dashboard> getDashboard(long userId) {

        CompletableFuture<Profile> profile =
                getProfile(userId);

        CompletableFuture<List<Order>> orders =
                getOrders(userId);

        CompletableFuture<List<Payment>> payments =
                getPayments(userId);

        CompletableFuture<List<Product>> recommendations =
                getRecommendations(userId);

        return CompletableFuture.allOf(
                        profile,
                        orders,
                        payments,
                        recommendations
                )
                .thenApply(v -> new Dashboard(
                        profile.join(),
                        orders.join(),
                        payments.join(),
                        recommendations.join()
                ));
    }

    // Mock service methods for interview practice.
    private CompletableFuture<Profile> getProfile(long userId) {
        return CompletableFuture.completedFuture(
                new Profile(userId, "Nirbhay"));
    }

    private CompletableFuture<List<Order>> getOrders(long userId) {
        return CompletableFuture.completedFuture(List.of());
    }

    private CompletableFuture<List<Payment>> getPayments(long userId) {
        return CompletableFuture.completedFuture(List.of());
    }

    private CompletableFuture<List<Product>> getRecommendations(long userId) {
        return CompletableFuture.completedFuture(List.of());
    }

    record Profile(long id, String name) {}
    record Order(long id) {}
    record Payment(long id) {}
    record Product(long id) {}

    record Dashboard(
            Profile profile,
            List<Order> orders,
            List<Payment> payments,
            List<Product> recommendations) {}
}
```

### Interview explanation

```text
1. Start independent operations concurrently.
2. allOf() creates one coordination future.
3. Wait for all operations to complete.
4. Read results from the original futures.
5. Build the final response.
```

---

# 33. Error Handling With Dashboard ⭐⭐⭐⭐⭐

If all components are mandatory:

```java
allOf(...)
```

can fail the whole dashboard when one future fails.

If some components are optional, recover individually:

```java
CompletableFuture<List<Product>> recommendations =
        getRecommendations(userId)
                .exceptionally(ex -> List.of());
```

Then the dashboard can still be built.

This is a very important production distinction:

```text
Mandatory dependency → failure may fail aggregate
Optional dependency   → recover with fallback
```

---

# 34. Practice — Partial Failure Recovery ⭐⭐⭐⭐⭐

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class PartialFailureDemo {

    public static void main(String[] args) {

        CompletableFuture<String> profile =
                CompletableFuture.completedFuture("Profile");

        CompletableFuture<String> orders =
                CompletableFuture.failedFuture(
                        new RuntimeException("Orders unavailable"));

        CompletableFuture<String> recommendations =
                CompletableFuture.completedFuture("Recommendations");

        CompletableFuture<String> safeOrders = orders
                .exceptionally(ex -> "Orders unavailable - fallback");

        CompletableFuture<Void> all = CompletableFuture.allOf(
                profile,
                safeOrders,
                recommendations
        );

        all.join();

        List<String> result = List.of(
                profile.join(),
                safeOrders.join(),
                recommendations.join()
        );

        System.out.println(result);
    }
}
```

---

# 35. Interview Question — Does `allOf()` Execute Tasks? ⭐⭐⭐⭐⭐

### Answer

> **No. `allOf()` coordinates the supplied `CompletableFuture` instances. The actual asynchronous tasks are normally started when those futures are created, for example through `supplyAsync()` or another async operation.**

Example:

```java
CompletableFuture<String> f1 =
        CompletableFuture.supplyAsync(task1);

CompletableFuture<String> f2 =
        CompletableFuture.supplyAsync(task2);

CompletableFuture.allOf(f1, f2);
```

The tasks were already submitted by `supplyAsync()`.

---

# 36. Interview Question — Does `anyOf()` Cancel Other Futures? ⭐⭐⭐⭐⭐

### Answer

> **No. `anyOf()` only completes when one supplied future completes. It does not automatically cancel the remaining futures. If the remaining work should be stopped, cancellation must be designed explicitly, and cancellation may not interrupt arbitrary underlying work.**

---

# 37. Interview Question — `allOf()` vs `thenCombine()` ⭐⭐⭐⭐⭐

### `thenCombine()`

Best for combining a small number of typed results:

```java
futureA.thenCombine(
        futureB,
        (a, b) -> combine(a, b)
);
```

### `allOf()`

Useful when coordinating many futures:

```java
CompletableFuture.allOf(
        future1,
        future2,
        future3,
        future4
);
```

Memory:

```text
Small typed combination → thenCombine
Many futures coordination → allOf
First completion race → anyOf
```

---

# 38. Interview Question — Why Not Just Use `anyOf()` for First Success? ⭐⭐⭐⭐⭐

Because:

```text
anyOf = first completion
```

not:

```text
first successful completion
```

A failed future can complete first and cause the combined future to fail.

For first-success semantics, design explicit failure filtering/recovery and consider cancellation/timeout/resource implications.

---

# 39. Complexity Discussion ⭐⭐⭐⭐

For N independent futures:

```text
Creating/submitting N tasks → depends on executor/workload
Coordination with allOf()   → O(N) references
Collecting N results        → O(N)
```

The important performance benefit is not that `allOf()` makes work magically faster.

The benefit comes from **starting independent operations concurrently** and then coordinating their completion.

---

# 40. 🏆 COMPLETE INTERVIEW PRACTICE CODE

This is the section you should practice writing from memory.

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

public class AllOfAnyOfInterviewPractice {

    public static void main(String[] args) {

        // ==============================
        // PART 1: allOf()
        // ==============================

        CompletableFuture<String> user =
                service("User", 700);

        CompletableFuture<String> orders =
                service("Orders", 1000);

        CompletableFuture<String> payments =
                service("Payments", 500);

        CompletableFuture<Void> all =
                CompletableFuture.allOf(
                        user,
                        orders,
                        payments
                );

        all.join();

        List<String> allResults = List.of(
                user.join(),
                orders.join(),
                payments.join()
        );

        System.out.println("ALL RESULTS = " + allResults);

        // ==============================
        // PART 2: anyOf()
        // ==============================

        CompletableFuture<String> serviceA =
                service("Service-A", 1200);

        CompletableFuture<String> serviceB =
                service("Service-B", 400);

        CompletableFuture<String> serviceC =
                service("Service-C", 800);

        Object fastest = CompletableFuture
                .anyOf(
                        serviceA,
                        serviceB,
                        serviceC
                )
                .join();

        System.out.println("FASTEST = " + fastest);
    }

    static CompletableFuture<String> service(
            String name,
            long delay) {

        return CompletableFuture.supplyAsync(() -> {
            sleep(delay);
            return name + " response";
        });
    }

    static void sleep(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

### Practice without looking

First write:

```java
CompletableFuture.allOf(...)
```

Then answer from memory:

```text
Q1. What does allOf() return?
Q2. How do you get individual results?
Q3. Does allOf() return List<T>?
Q4. What happens if one future fails?
Q5. What does anyOf() return?
Q6. Does anyOf() mean first successful result?
Q7. Are losing futures automatically cancelled?
Q8. When would you choose allOf vs thenCombine?
```

---

# 41. 2-Minute Interview Answer 🏆

> **"`CompletableFuture.allOf()` is used when I need to coordinate multiple independent asynchronous operations and wait until all of them complete. It returns `CompletableFuture<Void>`, so it doesn't directly aggregate the results. After `allOf().join()` completes, I retrieve the results from the original futures, often using a stream to collect them into a list. If one supplied future completes exceptionally, the combined `allOf` future completes exceptionally as well, although the other tasks can still complete independently. `CompletableFuture.anyOf()` is different: it completes when the first supplied future completes and returns `CompletableFuture<Object>`. Importantly, it means first completion, not first successful completion, and it does not automatically cancel the remaining futures. I use `allOf` for parallel aggregation and `anyOf` for race-style scenarios such as redundant providers, while considering timeouts, cancellation, side effects and resource usage."**

---

# 42. 30-Second Hinglish Answer

> **"`allOf()` tab use karta hoon jab multiple independent CompletableFutures ka result chahiye aur sabke complete hone ka wait karna hai. `allOf()` `CompletableFuture<Void>` return karta hai, isliye results original futures se `join()` karke lene padte hain. `anyOf()` first completed future ka result deta hai aur `Object` return karta hai. Important point: `anyOf()` first successful nahi, first completed future ko choose karta hai, aur baaki futures automatically cancel nahi hote. Simple memory: `ALL = sabka wait`, `ANY = first completion`."**

---

# 43. Common Interview Mistakes 🚨

### ❌ Mistake 1

```java
List<String> result = CompletableFuture.allOf(...)
```

Wrong because `allOf()` returns:

```java
CompletableFuture<Void>
```

### ❌ Mistake 2

Saying:

> `anyOf()` gives first successful result.

Wrong.

It gives the first **completed** result, including exceptional completion.

### ❌ Mistake 3

Saying:

> `anyOf()` cancels all other tasks.

Wrong.

### ❌ Mistake 4

Assuming `allOf()` makes tasks parallel.

The individual futures need to be started asynchronously/independently first.

### ❌ Mistake 5

Calling `join()` sequentially without understanding when the futures were started.

---

# 44. Quick Revision 🧠

```text
allOf(f1, f2, f3)
        ↓
wait for ALL
        ↓
CompletableFuture<Void>
        ↓
original futures → join()
```

```text
anyOf(f1, f2, f3)
        ↓
first completion
        ↓
CompletableFuture<Object>
        ↓
winner result
```

### Golden Rules

```text
ALL = coordination of all futures
ANY = race on first completion

allOf does NOT aggregate results automatically
anyOf does NOT mean first success
neither automatically cancels other tasks
```

---

# 45. 💻 Practice Checklist

- [ ] Write basic `allOf()` from memory
- [ ] Explain why `allOf()` returns `Void`
- [ ] Retrieve original future results after `allOf().join()`
- [ ] Convert `allOf()` results into `List<T>`
- [ ] Practice `anyOf()`
- [ ] Explain why `anyOf()` returns `Object`
- [ ] Demonstrate first completion
- [ ] Demonstrate first completion as failure
- [ ] Explain why `anyOf()` does not mean first success
- [ ] Explain cancellation behavior
- [ ] Use `allOf()` with custom executor
- [ ] Add timeout to `allOf()`
- [ ] Handle partial failures
- [ ] Build dashboard aggregation scenario
- [ ] Compare `allOf()` vs `thenCombine()`
- [ ] Explain production race considerations
- [ ] Write complete interview code without looking
- [ ] Give the 2-minute interview answer
- [ ] Give the 30-second Hinglish answer

---

## Navigation

[← 8.33 — Async Execution & Custom Executors](../33-Async-Execution-and-Custom-Executors/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.35 — Fork/Join Framework**