# 8.32 — Exception Handling in `CompletableFuture`

> **Goal:** Handle failures correctly in asynchronous pipelines without losing exceptions, blocking unnecessarily, or returning misleading success responses.

---

## 1. Core Mental Model ⭐⭐⭐⭐⭐

A `CompletableFuture` can complete in two broad ways:

```text
Success
  → result available

Failure
  → completed exceptionally
```

Exception handling is itself part of the asynchronous pipeline.

The most important APIs are:

```text
exceptionally()
handle()
whenComplete()
```

And for composing failure-aware pipelines:

```text
exceptionallyCompose()
```

---

# 2. Basic Exceptional Completion ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionalCompletionDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Database unavailable");
                });

        try {
            System.out.println(future.join());
        } catch (Exception e) {
            System.out.println("Future failed: " + e.getCause());
        }
    }
}
```

The exception completes the future exceptionally; it does not behave like an ordinary return value.

---

# 3. `join()` vs `get()` Exception Behavior ⭐⭐⭐⭐⭐

Both can observe an exceptional completion.

```java
future.join();
```

usually throws:

```text
CompletionException
```

while:

```java
future.get();
```

throws checked exceptions such as:

```text
ExecutionException
InterruptedException
```

Interview memory:

```text
join() → unchecked CompletionException
get()  → checked ExecutionException + InterruptedException
```

---

# 4. `exceptionally()` — Recover from Failure ⭐⭐⭐⭐⭐

Use `exceptionally()` when you want a fallback value if the pipeline fails.

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Service failed");
        }).exceptionally(ex -> "Fallback");
```

Result:

```text
Fallback
```

Mental model:

```text
Success → original result
Failure → fallback result
```

---

# 5. Practice — Basic `exceptionally()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionallyDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Payment service down");
                }).exceptionally(ex -> {
                    System.out.println("Error: " + ex.getMessage());
                    return "PAYMENT_UNAVAILABLE";
                });

        System.out.println(future.join());
    }
}
```

---

# 6. Important: `exceptionally()` Is a Recovery Stage ⭐⭐⭐⭐⭐

If the original stage succeeds:

```java
CompletableFuture.completedFuture("SUCCESS")
        .exceptionally(ex -> "FALLBACK");
```

returns:

```text
SUCCESS
```

The fallback is used only when the upstream stage completes exceptionally.

---

# 7. `exceptionally()` After a Pipeline ⭐⭐⭐⭐⭐

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "java")
                .thenApply(String::toUpperCase)
                .thenApply(value -> {
                    throw new RuntimeException("Processing failed");
                })
                .exceptionally(ex -> "DEFAULT");
```

Result:

```text
DEFAULT
```

Normal stages after the failure are skipped unless a recovery stage changes the pipeline back to successful completion.

---

# 8. `handle()` — Handle Both Success and Failure ⭐⭐⭐⭐⭐

Use `handle()` when the next stage must receive either:

```text
result + exception
```

Its function receives:

```java
(result, exception) -> ...
```

Example:

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "SUCCESS")
                .handle((result, ex) -> {
                    if (ex != null) {
                        return "FAILED";
                    }
                    return result;
                });
```

---

# 9. Practice — `handle()` on Success ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class HandleSuccessDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.completedFuture("SUCCESS")
                        .handle((result, ex) -> {
                            if (ex != null) {
                                return "FAILED";
                            }
                            return "Result = " + result;
                        });

        System.out.println(future.join());
    }
}
```

---

# 10. Practice — `handle()` on Failure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class HandleFailureDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Service failed");
                }).handle((result, ex) -> {
                    if (ex != null) {
                        return "Fallback because: " + ex.getMessage();
                    }
                    return result;
                });

        System.out.println(future.join());
    }
}
```

---

# 11. `exceptionally()` vs `handle()` ⭐⭐⭐⭐⭐

| API | Success input | Failure input | Can recover with value? |
|---|---|---|---|
| `exceptionally()` | No | Exception | Yes |
| `handle()` | Result | Exception | Yes |
| `whenComplete()` | Result | Exception | No — normally observation only |

Memory trick:

```text
exceptionally → only failure recovery
handle        → success + failure transformation
whenComplete  → observe completion
```

---

# 12. `whenComplete()` — Observe Completion ⭐⭐⭐⭐⭐

Use `whenComplete()` for side effects such as:

```text
logging
metrics
tracing
cleanup/diagnostics
```

Example:

```java
future.whenComplete((result, ex) -> {
    System.out.println("Completed");
});
```

It does not normally convert a failed future into a successful one.

---

# 13. Practice — `whenComplete()` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class WhenCompleteDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> "Order-101")
                        .whenComplete((result, ex) -> {
                            if (ex != null) {
                                System.out.println("Order failed: " + ex);
                            } else {
                                System.out.println("Order completed: " + result);
                            }
                        });

        System.out.println(future.join());
    }
}
```

---

# 14. `whenComplete()` Does Not Mean Recovery ⭐⭐⭐⭐⭐

This is a common interview trap.

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Failure");
        }).whenComplete((result, ex) ->
                System.out.println("Logging failure"));
```

The future is still exceptional.

If you need a fallback value, use:

```java
exceptionally()
```

or another recovery strategy.

---

# 15. Practice — Observe Failure Without Recovering ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class WhenCompleteFailureDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Database error");
                }).whenComplete((result, ex) -> {
                    if (ex != null) {
                        System.out.println("LOG: " + ex);
                    }
                });

        try {
            future.join();
        } catch (Exception e) {
            System.out.println("Still failed: " + e.getCause());
        }
    }
}
```

---

# 16. `handle()` vs `whenComplete()` ⭐⭐⭐⭐⭐

The key distinction:

```text
handle()
→ inspect result/exception
→ transform into a new result
```

```text
whenComplete()
→ inspect result/exception
→ normally preserve the original outcome
```

Example:

```java
future.handle((result, ex) -> "new result");
```

can turn the stage into a successful result.

Whereas:

```java
future.whenComplete((result, ex) -> log(ex));
```

is primarily for observation.

---

# 17. Exception Propagation in Chains ⭐⭐⭐⭐⭐

Consider:

```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "java")
                .thenApply(value -> {
                    throw new RuntimeException("Step 2 failed");
                })
                .thenApply(String::toUpperCase)
                .exceptionally(ex -> "RECOVERED");
```

Flow:

```text
supplyAsync
    ↓ success
thenApply
    ↓ exception
next thenApply skipped normally
    ↓
exceptionally
    ↓
RECOVERED
```

---

# 18. Practice — Exception Propagation ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionPropagationDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> "java")
                        .thenApply(value -> {
                            System.out.println("Step 1: " + value);
                            throw new RuntimeException("Step 2 failed");
                        })
                        .thenApply(value -> {
                            System.out.println("Step 3: " + value);
                            return value.toUpperCase();
                        })
                        .exceptionally(ex -> {
                            System.out.println("Recovering: " + ex);
                            return "RECOVERED";
                        });

        System.out.println(future.join());
    }
}
```

---

# 19. Recovery Changes the Pipeline ⭐⭐⭐⭐⭐

After recovery:

```java
.exceptionally(ex -> "DEFAULT")
.thenApply(String::toUpperCase)
```

the later `thenApply()` can run because the recovery stage returned a normal value.

Flow:

```text
Failure
  ↓
exceptionally
  ↓ "DEFAULT"
thenApply
  ↓ "DEFAULT"
"DEFAULT"
```

---

# 20. Practice — Recovery Then Continue ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class RecoveryThenContinueDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.<String>supplyAsync(() -> {
                    throw new RuntimeException("Failure");
                })
                .exceptionally(ex -> "fallback")
                .thenApply(String::toUpperCase);

        System.out.println(future.join());
    }
}
```

Output:

```text
FALLBACK
```

---

# 21. `exceptionallyCompose()` — Async Recovery ⭐⭐⭐⭐⭐

When recovery itself requires an asynchronous operation, `exceptionallyCompose()` is useful.

Conceptually:

```text
Failure
  ↓
async fallback operation
  ↓
Future<R>
```

Example:

```java
CompletableFuture<String> result =
        primaryCall()
                .exceptionallyCompose(ex -> fallbackCall());
```

This avoids creating an unnecessary nested future.

---

# 22. Practice — Async Fallback ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionallyComposeDemo {

    static CompletableFuture<String> primaryCall() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Primary service down");
        });
    }

    static CompletableFuture<String> fallbackCall() {
        return CompletableFuture.supplyAsync(() -> "Fallback service response");
    }

    public static void main(String[] args) {
        CompletableFuture<String> result =
                primaryCall()
                        .exceptionallyCompose(ex -> fallbackCall());

        System.out.println(result.join());
    }
}
```

---

# 23. `exceptionally()` vs `exceptionallyCompose()` ⭐⭐⭐⭐⭐

```text
exceptionally
→ failure → normal fallback value

exceptionallyCompose
→ failure → asynchronous fallback Future
```

Example:

```java
.exceptionally(ex -> "cached-value")
```

versus:

```java
.exceptionallyCompose(ex -> loadFromBackupService())
```

---

# 24. Practice — Primary + Cache Fallback ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class CacheFallbackDemo {

    static CompletableFuture<String> database() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("DB unavailable");
        });
    }

    static CompletableFuture<String> cache() {
        return CompletableFuture.supplyAsync(() -> "Cached user");
    }

    public static void main(String[] args) {
        CompletableFuture<String> user =
                database().exceptionallyCompose(ex -> cache());

        System.out.println(user.join());
    }
}
```

Production pattern:

```text
Primary DB
   ↓ failure
Cache / backup service
   ↓
Response
```

---

# 25. Exception Wrapping ⭐⭐⭐⭐⭐

Exceptions observed through `CompletableFuture` APIs can be wrapped.

With:

```java
future.join()
```

you commonly see:

```text
CompletionException
    └── cause
```

Therefore, for diagnostics:

```java
try {
    future.join();
} catch (CompletionException e) {
    Throwable cause = e.getCause();
    System.out.println(cause);
}
```

Do not blindly assume `e.getMessage()` is the root cause.

---

# 26. Practice — Inspect Root Cause ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionException;

public class RootCauseDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new IllegalStateException("Actual failure");
                });

        try {
            future.join();
        } catch (CompletionException e) {
            System.out.println("Wrapper: " + e.getClass().getSimpleName());
            System.out.println("Root cause: " + e.getCause());
        }
    }
}
```

---

# 27. Cancellation Is Also Exceptional Completion ⭐⭐⭐⭐⭐

A cancelled `CompletableFuture` is completed exceptionally with cancellation semantics.

```java
CompletableFuture<String> future =
        new CompletableFuture<>();

future.cancel(true);
```

Downstream stages can observe the abnormal completion.

Do not assume `cancel(true)` forcibly interrupts arbitrary asynchronous work in every executor/context.

---

# 28. Practice — Cancellation Observation ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class CancellationDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                new CompletableFuture<>();

        future.whenComplete((result, ex) ->
                System.out.println("Completed: " +
                        future.isCancelled()));

        future.cancel(true);
    }
}
```

---

# 29. Timeout as a Failure Path ⭐⭐⭐⭐⭐

Timeout APIs can introduce exceptional completion.

Example:

```java
CompletableFuture<String> future =
        slowService()
                .orTimeout(1, TimeUnit.SECONDS);
```

If the timeout expires before normal completion, the future completes exceptionally.

You can then recover:

```java
.orTimeout(1, TimeUnit.SECONDS)
.exceptionally(ex -> "TIMEOUT_FALLBACK");
```

---

# 30. Practice — Timeout + Recovery ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class TimeoutRecoveryDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    try {
                        Thread.sleep(2_000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException(e);
                    }
                    return "SUCCESS";
                })
                .orTimeout(500, TimeUnit.MILLISECONDS)
                .exceptionally(ex -> "TIMEOUT_FALLBACK");

        System.out.println(future.join());
    }
}
```

---

# 31. `completeOnTimeout()` vs `orTimeout()` ⭐⭐⭐⭐⭐

```text
orTimeout
→ timeout → exceptional completion

completeOnTimeout
→ timeout → normal completion with fallback value
```

Example:

```java
future.orTimeout(1, TimeUnit.SECONDS);
```

versus:

```java
future.completeOnTimeout("DEFAULT", 1, TimeUnit.SECONDS);
```

This distinction is important in production API design.

---

# 32. Practice — Two Timeout Strategies ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class TimeoutStrategiesDemo {

    public static void main(String[] args) {
        CompletableFuture<String> exceptionalTimeout =
                new CompletableFuture<String>()
                        .orTimeout(100, TimeUnit.MILLISECONDS);

        CompletableFuture<String> valueTimeout =
                new CompletableFuture<String>()
                        .completeOnTimeout("DEFAULT", 100,
                                TimeUnit.MILLISECONDS);

        try {
            exceptionalTimeout.join();
        } catch (Exception e) {
            System.out.println("orTimeout -> exceptional");
        }

        System.out.println("completeOnTimeout -> " +
                valueTimeout.join());
    }
}
```

---

# 33. Error Handling at the Right Boundary ⭐⭐⭐⭐⭐

Avoid putting one giant `exceptionally()` at the very end without thinking about error semantics.

For example:

```text
Payment failure
Inventory failure
Recommendation failure
```

may need different policies:

```text
Payment → fail request
Inventory → retry/fallback
Recommendations → optional empty list
```

Exception handling should reflect business meaning, not simply hide every exception.

---

# 34. Practice — Different Recovery Policies ⭐⭐⭐⭐⭐

```java
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class RecoveryPolicyDemo {

    static CompletableFuture<String> payment() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Payment failed");
        });
    }

    static CompletableFuture<List<String>> recommendations() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Recommendation failed");
        }).exceptionally(ex -> Collections.emptyList());
    }

    public static void main(String[] args) {
        try {
            payment().join();
        } catch (Exception e) {
            System.out.println("Payment failure should reach caller");
        }

        System.out.println(
                "Recommendations = " + recommendations().join());
    }
}
```

---

# 35. Logging Without Losing the Failure ⭐⭐⭐⭐⭐

A useful pattern is:

```java
.whenComplete((result, ex) -> {
    if (ex != null) {
        logFailure(ex);
    }
})
.exceptionally(ex -> fallback());
```

This separates:

```text
Observation / logging
        ↓
Recovery policy
```

Be careful not to log the same exception repeatedly at every stage and create noisy production logs.

---

# 36. Practice — Log Then Recover ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class LogThenRecoverDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Remote API failed");
                })
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        System.out.println("LOG: remote API failure");
                    }
                })
                .exceptionally(ex -> "Cached response");

        System.out.println(future.join());
    }
}
```

---

# 37. Exception Handling with `thenCompose()` ⭐⭐⭐⭐⭐

A failure in the dependent operation propagates into the composed future.

```java
CompletableFuture<String> result =
        loadUser()
                .thenCompose(user -> loadOrders(user))
                .exceptionally(ex -> "NO_ORDERS");
```

Both of these can fail:

```text
loadUser()
loadOrders(user)
```

The downstream recovery can handle the exceptional completion.

---

# 38. Practice — Dependent Call Failure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ComposeFailureDemo {

    static CompletableFuture<String> user() {
        return CompletableFuture.completedFuture("User-101");
    }

    static CompletableFuture<String> orders(String user) {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Orders service failed");
        });
    }

    public static void main(String[] args) {
        CompletableFuture<String> result =
                user()
                        .thenCompose(ComposeFailureDemo::orders)
                        .exceptionally(ex -> "ORDERS_UNAVAILABLE");

        System.out.println(result.join());
    }
}
```

---

# 39. Exception Handling with `thenCombine()` ⭐⭐⭐⭐⭐

If one of the combined futures completes exceptionally, the combined stage completes exceptionally rather than producing a normal combined result.

Example:

```text
User Future       → SUCCESS
Recommendation    → FAILURE
                         ↓
                    combined Future
                         ↓
                       FAILURE
```

You can recover at the combined boundary or design independent fallbacks before combining, depending on business semantics.

---

# 40. Practice — Independent Failure + Combine ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class CombineFailureDemo {

    public static void main(String[] args) {
        CompletableFuture<String> user =
                CompletableFuture.completedFuture("Nirbhay");

        CompletableFuture<String> recommendation =
                CompletableFuture.supplyAsync(() -> {
                    throw new RuntimeException("Recommendation failed");
                });

        CompletableFuture<String> result =
                user.thenCombine(recommendation,
                        (u, r) -> u + " + " + r)
                .exceptionally(ex ->
                        "User response without recommendations");

        System.out.println(result.join());
    }
}
```

---

# 41. Retry Is Not Built Into `CompletableFuture` ⭐⭐⭐⭐⭐

`CompletableFuture` provides composition and error-handling primitives, but a general-purpose retry policy is not automatically applied.

A retry can be modeled explicitly, for example with a recursive asynchronous function, a retry library, or application infrastructure.

Do not claim:

```text
exceptionally() = retry
```

It is recovery, not automatic retry.

---

# 42. Practice — Simple Explicit Retry ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicInteger;

public class ExplicitRetryDemo {

    static AtomicInteger attempts = new AtomicInteger();

    static CompletableFuture<String> call() {
        return CompletableFuture.supplyAsync(() -> {
            int attempt = attempts.incrementAndGet();
            System.out.println("Attempt " + attempt);

            if (attempt < 3) {
                throw new RuntimeException("Temporary failure");
            }
            return "SUCCESS";
        });
    }

    static CompletableFuture<String> retry(int remaining) {
        return call().exceptionallyCompose(ex -> {
            if (remaining <= 0) {
                return CompletableFuture.failedFuture(ex);
            }
            return retry(remaining - 1);
        });
    }

    public static void main(String[] args) {
        System.out.println(retry(2).join());
    }
}
```

This is intentionally simple; production retry needs limits, delay/backoff, classification of retryable errors, and careful resource management.

---

# 43. `failedFuture()` — Explicit Failed Stage ⭐⭐⭐⭐

When you need to return an already-failed `CompletableFuture`:

```java
return CompletableFuture.failedFuture(
        new IllegalStateException("Invalid request"));
```

This is useful in validation or asynchronous control flow.

---

# 44. Practice — Validation Failure ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class FailedFutureDemo {

    static CompletableFuture<String> validate(String value) {
        if (value == null || value.isBlank()) {
            return CompletableFuture.failedFuture(
                    new IllegalArgumentException("Value is required"));
        }

        return CompletableFuture.completedFuture(value);
    }

    public static void main(String[] args) {
        System.out.println(
                validate("")
                        .exceptionally(ex -> "INVALID_INPUT")
                        .join());
    }
}
```

---

# 45. Production Pattern — Service Boundary ⭐⭐⭐⭐⭐

A production API pipeline might look like:

```text
Request validation
      ↓
Primary async service
      ↓
whenComplete → metrics/logging
      ↓
exceptionally / exceptionallyCompose
      ↓
Business fallback
      ↓
Response
```

Important:

```text
technical exception
        ↓
business decision
        ↓
retry / fallback / fail
```

Do not blindly convert every exception into HTTP 200 or a successful business response.

---

# 46. Practice — Production-Style Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.CompletableFuture;

public class ProductionPipelineDemo {

    static CompletableFuture<String> callService() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Service unavailable");
        });
    }

    public static void main(String[] args) {
        CompletableFuture<String> response =
                callService()
                        .whenComplete((result, ex) -> {
                            if (ex != null) {
                                System.out.println("METRIC: service.failure");
                            }
                        })
                        .exceptionally(ex ->
                                "BUSINESS_FALLBACK");

        System.out.println(response.join());
    }
}
```

---

# 47. Common Interview Traps 🚨

```text
❌ exceptionally() always retries
❌ whenComplete() recovers exceptions
❌ handle() only handles exceptions
❌ join() throws checked ExecutionException
❌ completeOnTimeout() fails exceptionally
❌ every exception should be swallowed
❌ cancellation means arbitrary worker code is always interrupted
```

Correct:

```text
exceptionally()       → failure → fallback value
handle()              → success/failure → transformed result
whenComplete()        → observe outcome
exceptionallyCompose  → failure → async fallback
orTimeout()           → exceptional timeout
completeOnTimeout()   → normal fallback timeout
join()                → CompletionException observation
get()                 → checked exception handling
```

---

# 48. Interview Questions ⭐⭐⭐⭐⭐

### Q1. How do you handle exceptions in `CompletableFuture`?

> Common choices are `exceptionally()` for failure recovery, `handle()` when I need both result and exception to produce a new result, and `whenComplete()` for observing/logging completion without treating it as the recovery mechanism.

### Q2. `exceptionally()` vs `handle()`?

> `exceptionally()` runs for exceptional completion and provides a fallback value. `handle()` receives both the result and exception and can transform either success or failure into a new result.

### Q3. Does `whenComplete()` recover an exception?

> No. It is primarily an observation/side-effect stage. If the upstream stage failed, the resulting stage normally remains failed unless the callback itself introduces a different outcome.

### Q4. What is `exceptionallyCompose()`?

> It is useful when failure recovery itself is asynchronous and returns another `CompletionStage`, avoiding a nested future.

### Q5. What happens if a `thenApply()` stage throws?

> Its resulting stage completes exceptionally, and normal downstream stages are skipped until a recovery/handling stage processes that failure.

### Q6. `orTimeout()` vs `completeOnTimeout()`?

> `orTimeout()` completes exceptionally on timeout, while `completeOnTimeout()` completes normally with the supplied fallback value.

### Q7. Does `CompletableFuture` automatically retry failed operations?

> No. Retry must be explicitly designed or delegated to appropriate resilience infrastructure.

---

# 49. Senior-Level Scenario 🏆

### Question

> Payment is critical, but recommendations are optional. How would you design the async error handling?

### Good answer

```text
Payment
 → failure should propagate / fail the request

Recommendations
 → recover with empty recommendations
```

For example:

```java
CompletableFuture<Payment> payment = paymentService();

CompletableFuture<List<Recommendation>> recommendations =
        recommendationService()
                .exceptionally(ex -> Collections.emptyList());
```

Then combine according to the API's business semantics.

The key is **not every downstream failure should have the same recovery policy**.

---

# 50. 2-Minute Interview Answer 🏆

> **"For `CompletableFuture` exception handling, I mainly use `exceptionally`, `handle`, and `whenComplete`. `exceptionally` is for recovering from an exceptional completion with a fallback value. `handle` receives both the result and exception, so it is useful when I need to transform either success or failure into a new result. `whenComplete` is mainly for side effects such as logging, metrics, and tracing, and normally preserves the original outcome. If the fallback itself is asynchronous, I can use `exceptionallyCompose`. I also distinguish `orTimeout`, which causes exceptional completion, from `completeOnTimeout`, which supplies a normal fallback value. In production I don't swallow every exception; I classify failures into retryable, fallback-able, or fatal cases based on business semantics."**

---

# 51. Quick Revision ⭐⭐⭐⭐⭐

```text
                 CompletableFuture Failure Handling
                              │
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
 exceptionally              handle            whenComplete
        │                     │                     │
 failure only          success + failure       observe
        │                     │                     │
 fallback value         new transformed value    logging/metrics

                 Async failure recovery
                         ↓
                exceptionallyCompose
```

### Golden Rule 🧠

> **Recover with `exceptionally`, transform with `handle`, observe with `whenComplete`, and use `exceptionallyCompose` when the fallback is itself asynchronous.**

---

# 52. 💻 Practice Checklist

- [ ] Create an exceptionally completed future
- [ ] Compare `join()` and `get()` exception behavior
- [ ] Basic `exceptionally()`
- [ ] Recovery after a failed pipeline stage
- [ ] `handle()` on success
- [ ] `handle()` on failure
- [ ] `whenComplete()` on success
- [ ] `whenComplete()` on failure
- [ ] Prove `whenComplete()` does not normally recover
- [ ] Compare `exceptionally()` vs `handle()` vs `whenComplete()`
- [ ] Understand exception propagation
- [ ] Recover and continue the pipeline
- [ ] Practice `exceptionallyCompose()`
- [ ] Build async DB/cache fallback
- [ ] Inspect `CompletionException` root cause
- [ ] Understand cancellation as exceptional completion
- [ ] Practice `orTimeout()` recovery
- [ ] Compare `orTimeout()` vs `completeOnTimeout()`
- [ ] Design different recovery policies for different services
- [ ] Log failure without swallowing it accidentally
- [ ] Handle `thenCompose()` failure
- [ ] Handle `thenCombine()` failure
- [ ] Understand explicit retry vs recovery
- [ ] Practice `failedFuture()`
- [ ] Build production-style error pipeline
- [ ] Answer senior-level scenario aloud
- [ ] Give the 2-minute interview answer

---

## Navigation

[← 8.31 — `thenApply` / `thenCompose` / `thenCombine`](../31-thenApply-thenCompose-thenCombine/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.33 — Async Execution & Custom Executors**