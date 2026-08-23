# 8.9 — Queueing, Rejection Policies & Backpressure

> **Goal:** Understand how executor queues control task flow, what happens when a pool becomes saturated, and how rejection/backpressure protect a system from overload.

---

## 1. Why Queueing & Backpressure? ⭐⭐⭐⭐⭐

A concurrent system has two different rates:

```text
Producer rate  →  tasks arriving
Consumer rate  →  tasks being processed
```

If:

```text
Producer rate > Consumer rate
```

then pending work grows.

```text
Producer
   ↓
   ↓↓↓ tasks
┌──────────────┐
│ Work Queue   │  ← grows under overload
└──────────────┘
       ↓
┌──────────────┐
│ Worker Pool  │
└──────────────┘
       ↓
   Consumers
```

Eventually the system needs a policy:

```text
Queue full
   ↓
Can executor create more workers?
   ↓
If no → reject / shed / slow down work
```

This is where **rejection policies and backpressure** become important.

---

# 2. Queueing Mental Model ⭐⭐⭐⭐⭐

With a `ThreadPoolExecutor`:

```text
Task submitted
      ↓
Core workers available?
      ↓ no
Try queue
      ↓
Queue accepts?
   ↙       ↘
 YES       NO
  ↓         ↓
wait     max workers?
            ↓
        YES → new worker
            ↓
           NO
            ↓
         REJECT
```

### Memory Trick

> **Core → Queue → Max → Reject**

The queue is therefore not just storage; it participates in the executor's concurrency and overload behavior.

---

# 3. Why Bounded Queues Matter ⭐⭐⭐⭐⭐

A bounded queue gives an explicit limit:

```java
new ArrayBlockingQueue<>(100)
```

Without a meaningful bound, producers can continue submitting work while consumers fall behind.

### Risk of uncontrolled queue growth

```text
Traffic spike
    ↓
More tasks
    ↓
Queue grows
    ↓
Memory pressure
    ↓
Latency increases
    ↓
Possible OOM / cascading failure
```

### Key principle

> **A queue can absorb a burst, but it cannot permanently compensate for a consumer that is slower than the producer.**

---

# 4. Practice — Bounded Queue ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class BoundedQueueExample {
    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2)
        );

        for (int i = 1; i <= 8; i++) {
            int taskId = i;

            try {
                executor.execute(() -> {
                    System.out.println("Running task " + taskId
                            + " on " + Thread.currentThread().getName());
                    sleep(2000);
                });

                System.out.println(
                        "Task " + taskId
                                + " | pool=" + executor.getPoolSize()
                                + " | queue=" + executor.getQueue().size());

            } catch (RejectedExecutionException e) {
                System.out.println("Rejected task " + taskId);
            }
        }

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Use this to observe how a bounded queue changes behavior when the executor becomes saturated.

---

# 5. Queue Types and Their Behavior ⭐⭐⭐⭐⭐

| Queue | Main characteristic | Typical implication |
|---|---|---|
| `ArrayBlockingQueue` | Bounded | Explicit capacity/backpressure point |
| `LinkedBlockingQueue` | Optionally bounded | Can be bounded; default constructor has very large capacity |
| `SynchronousQueue` | No normal stored capacity | Direct handoff between producer and worker |
| `PriorityBlockingQueue` | Priority ordered, unbounded | Priority affects queued-task order |

### Important

The queue affects **when** the executor creates workers beyond the core size and **when** rejection can happen.

---

# 6. Rejection ⭐⭐⭐⭐⭐

A task can be rejected when:

```text
Executor is running
       +
Queue cannot accept task
       +
Worker count has reached maximum
```

Java provides four standard handlers:

```java
AbortPolicy
CallerRunsPolicy
DiscardPolicy
DiscardOldestPolicy
```

---

# 7. `AbortPolicy` ⭐⭐⭐⭐⭐

```java
new ThreadPoolExecutor.AbortPolicy()
```

Default behavior of `ThreadPoolExecutor`.

It throws:

```java
RejectedExecutionException
```

### Practice Code

```java
import java.util.concurrent.*;

public class AbortBackpressureExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThreadPoolExecutor.AbortPolicy()
        );

        try {
            executor.execute(() -> sleep(3000));
            executor.execute(() -> System.out.println("Queued task"));
            executor.execute(() -> System.out.println("Third task"));
        } catch (RejectedExecutionException e) {
            System.out.println("Task rejected: " + e.getClass().getSimpleName());
        } finally {
            executor.shutdown();
        }
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### When useful

When rejection must be explicit and the caller can handle it.

For critical systems, rejection should normally become an observable business/operational event rather than being silently ignored.

---

# 8. `CallerRunsPolicy` ⭐⭐⭐⭐⭐

```java
new ThreadPoolExecutor.CallerRunsPolicy()
```

When saturated and the executor is still running, the submitting thread executes the task itself.

```text
Producer
   ↓
Executor full
   ↓
Producer executes task
   ↓
Producer becomes slower
```

This can provide a simple form of **producer-side slowdown**.

### Practice Code

```java
import java.util.concurrent.*;

public class CallerRunsBackpressureExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        for (int i = 1; i <= 6; i++) {
            int taskId = i;

            System.out.println("Submitting task " + taskId);

            executor.execute(() -> {
                System.out.println("Task " + taskId + " running on "
                        + Thread.currentThread().getName());
                sleep(1000);
            });
        }

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Interview point

> `CallerRunsPolicy` can slow a fast producer because the producer performs work when the executor is saturated, but it is not a complete backpressure architecture.

---

# 9. `DiscardPolicy` ⭐⭐⭐

```java
new ThreadPoolExecutor.DiscardPolicy()
```

The rejected task is silently discarded.

### Practice Code

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        1,
        1,
        0,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1),
        new ThreadPoolExecutor.DiscardPolicy()
);

executor.execute(() -> sleep(3000));
executor.execute(() -> System.out.println("Queued"));
executor.execute(() -> System.out.println("This may be discarded"));

executor.shutdown();
```

### Warning ⚠️

Do not use this for work that cannot safely be lost.

Examples where silent loss is generally dangerous:

- Payment state changes
- Order processing
- Audit records
- Security events

unless another durable mechanism guarantees recovery.

---

# 10. `DiscardOldestPolicy` ⭐⭐⭐

```java
new ThreadPoolExecutor.DiscardOldestPolicy()
```

When saturated:

```text
Remove oldest queued task
        ↓
Retry submission
```

### Practice Code

```java
import java.util.concurrent.*;

public class DiscardOldestExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2),
                new ThreadPoolExecutor.DiscardOldestPolicy()
        );

        executor.execute(() -> sleep(3000));
        executor.execute(() -> System.out.println("Task A"));
        executor.execute(() -> System.out.println("Task B"));
        executor.execute(() -> System.out.println("Task C"));

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Warning

Discarding the oldest item can violate ordering and business correctness.

---

# 11. Rejection Policy Comparison ⭐⭐⭐⭐⭐

| Policy | Result | Producer slowed? | Task lost? |
|---|---|---:|---:|
| `AbortPolicy` | Exception | No, unless caller handles it | No automatic silent loss |
| `CallerRunsPolicy` | Caller executes | Often yes | No, if caller executes successfully |
| `DiscardPolicy` | Drop task | No | Yes |
| `DiscardOldestPolicy` | Drop oldest queued task | No | Yes |

### Memory Trick

```text
Abort       → throw
CallerRuns  → caller works
Discard     → drop
Oldest      → drop oldest
```

---

# 12. Backpressure — What Does It Actually Mean? ⭐⭐⭐⭐⭐

**Backpressure** means slowing or limiting producers when consumers cannot keep up.

```text
Fast Producer
     ↓
  overload
     ↓
Backpressure
     ↓
Producer slows / work is rejected / work is buffered elsewhere
```

It is not simply "having a queue."

A queue can buffer a burst, while backpressure controls what happens when buffering is insufficient.

---

# 13. Backpressure Strategies ⭐⭐⭐⭐⭐

### Strategy 1 — Bounded Queue

```java
new ArrayBlockingQueue<>(100)
```

Limits queued tasks.

### Strategy 2 — `CallerRunsPolicy`

Makes the producer perform work under saturation.

### Strategy 3 — Explicit Rejection

```java
try {
    executor.execute(task);
} catch (RejectedExecutionException e) {
    // retry, return error, persist, or route elsewhere
}
```

### Strategy 4 — Rate Limiting

Limit how quickly producers submit work.

### Strategy 5 — Load Shedding

Intentionally reject lower-priority work when the system is overloaded.

### Strategy 6 — Durable Messaging

Move work to a durable broker when asynchronous buffering and recovery are required.

### Strategy 7 — Admission Control

Accept work only when the service has enough capacity.

---

# 14. Practice — Producer vs Consumer Rate ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ProducerConsumerBackpressureExample {
    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                2,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(3),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        for (int i = 1; i <= 15; i++) {
            int taskId = i;

            long start = System.currentTimeMillis();

            executor.execute(() -> {
                System.out.println("Processing " + taskId
                        + " on " + Thread.currentThread().getName());
                sleep(500);
            });

            long elapsed = System.currentTimeMillis() - start;
            System.out.println("Submitted " + taskId
                    + " in " + elapsed + " ms"
                    + " | queue=" + executor.getQueue().size());
        }

        executor.shutdown();
        executor.awaitTermination(15, TimeUnit.SECONDS);
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Observe that submission can take longer when the caller has to execute tasks.

---

# 15. Queue Capacity vs Latency ⭐⭐⭐⭐⭐

A larger queue is not automatically better.

```text
Small queue
 → earlier rejection
 → lower waiting time under overload

Large queue
 → absorbs more burst
 → potentially larger waiting time
 → delayed failure
```

### Important concept

> **Throughput and latency are different goals.**

A huge queue can make a system appear healthy while requests wait for a very long time.

---

# 16. Queueing Little's Law Connection ⭐⭐⭐⭐

A useful performance relationship is:

```text
L = λ × W
```

Where:

- `L` = average number of items in the system
- `λ` = average arrival rate
- `W` = average time an item spends in the system

### Example

If a system processes approximately:

```text
100 tasks/second
```

and average end-to-end time is:

```text
0.5 seconds
```

then:

```text
L = 100 × 0.5 = 50 tasks
```

This is a useful reasoning tool, not a promise that every thread pool will behave exactly this way under bursty workloads.

---

# 17. Avoid Unbounded Queues for Strict Resource Control ⭐⭐⭐⭐⭐

This pattern is risky when producers can outrun consumers:

```java
Executors.newFixedThreadPool(10);
```

The convenience factory uses an unbounded `LinkedBlockingQueue` in the standard implementation.

Consequences can include:

```text
Tasks accumulate
      ↓
Memory usage grows
      ↓
Latency grows
      ↓
Overload becomes harder to detect
```

For explicit overload control:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,
        8,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

---

# 18. Real-World Scenario — HTTP Request Processing ⭐⭐⭐⭐⭐

Suppose:

```text
Incoming HTTP requests
          ↓
Service executor
          ↓
Slow downstream API
```

If requests arrive faster than the downstream API can complete:

```text
Requests
  ↓
Queue grows
  ↓
Timeouts
  ↓
Retries
  ↓
More requests
  ↓
Cascading overload
```

### Better design considerations

- Bounded concurrency
- Bounded queues
- Timeouts
- Rate limits
- Circuit breakers
- Retry budgets
- Load shedding
- Observability

> **Retries can amplify overload**, so retry behavior must be designed together with capacity and backpressure.

---

# 19. Real-World Scenario — Payment Processing ⭐⭐⭐⭐⭐

For payment-related work, silently dropping a task is generally unacceptable.

Bad:

```java
new ThreadPoolExecutor.DiscardPolicy()
```

A better approach may involve:

```text
Request
  ↓
Validate
  ↓
Bounded processing
  ↓
If overloaded
  ├── explicit failure
  ├── durable queue
  └── retry/recovery workflow
```

The exact choice depends on business guarantees and architecture.

---

# 20. Priority-Based Load Shedding ⭐⭐⭐⭐

Not all work has equal business value.

Example:

```text
HIGH    → payment confirmation
MEDIUM  → notification
LOW     → analytics enrichment
```

Under overload, the system may prefer rejecting low-priority work first.

Possible building blocks:

- Priority queues
- Separate executors
- Separate queues
- Admission control
- Explicit load-shedding rules

Do not rely on `DiscardOldestPolicy` as a generic priority mechanism; it discards based on queue position, not business priority.

---

# 21. Separate Pools for Isolation ⭐⭐⭐⭐⭐

A single pool can create resource contention:

```text
               Shared Pool
              /           \
       Slow API           Fast task
           ↓
    Slow tasks occupy workers
           ↓
    Fast tasks delayed
```

Separate pools can provide bulkhead-style isolation:

```text
API Pool       → external API calls
DB Pool        → database work
CPU Pool       → CPU-heavy tasks
```

But too many pools can also increase complexity and resource usage.

---

# 22. Observability ⭐⭐⭐⭐⭐

Monitor:

```java
executor.getPoolSize();
executor.getActiveCount();
executor.getQueue().size();
executor.getLargestPoolSize();
executor.getCompletedTaskCount();
```

Important production metrics include:

- Queue depth
- Queue wait time
- Task execution time
- Rejection count
- Active workers
- Pool utilization
- Task timeout count
- Downstream latency
- Error rate

### Alerting idea

```text
Queue depth ↑
+ execution latency ↑
+ rejection count ↑
        ↓
Potential overload
```

---

# 23. Common Mistakes ❌

### Mistake 1 — "A queue solves overload."

No. It only buffers work temporarily.

### Mistake 2 — "Make the queue huge."

A huge queue can turn overload into high latency and memory pressure.

### Mistake 3 — "Use `DiscardPolicy` to avoid errors."

You may silently lose business-critical work.

### Mistake 4 — "CallerRunsPolicy is always best."

It can make the request thread perform expensive work and may increase request latency.

### Mistake 5 — "Increase max threads whenever queue grows."

The bottleneck may be CPU, DB connections, downstream APIs or another resource.

### Mistake 6 — "Retry every rejected task immediately."

Immediate retries can create a retry storm and make overload worse.

### Mistake 7 — "All workloads should share one pool."

Different workloads may require isolation.

---

# 24. Practice — Handle Rejection Explicitly ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ExplicitRejectionHandlingExample {
    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                1,
                1,
                0,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThreadPoolExecutor.AbortPolicy()
        );

        Runnable task = () -> sleep(2000);

        for (int i = 1; i <= 5; i++) {
            try {
                executor.execute(task);
                System.out.println("Accepted task " + i);
            } catch (RejectedExecutionException e) {
                System.out.println("Rejected task " + i
                        + " — handle overload here");
            }
        }

        executor.shutdown();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Possible real handling:

```text
Rejected
   ↓
Return controlled error
OR
Persist for later
OR
Route to durable queue
OR
Drop only if business-approved
```

---

# 25. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. What is backpressure?

> Backpressure is a mechanism that limits or slows producers when consumers cannot keep up, preventing uncontrolled buildup of work.

### Q2. Why use a bounded queue?

> It places an explicit upper limit on queued work and makes overload behavior visible and controllable.

### Q3. What happens when a `ThreadPoolExecutor` is saturated?

> Once core workers are active, the queue is full, and worker count has reached maximum, the configured rejection policy handles the new task.

### Q4. Why can an unbounded queue be dangerous?

> It can allow queued work and latency to grow without a meaningful bound, causing memory pressure and delayed overload detection.

### Q5. Explain `CallerRunsPolicy`.

> The submitting thread executes the task when the pool is saturated and still running, which can slow the producer.

### Q6. Is `CallerRunsPolicy` true backpressure?

> It is a simple form of producer slowdown, but complete backpressure usually involves admission control, bounded resources, rate limiting and appropriate failure handling.

### Q7. Why not use `DiscardPolicy` for everything?

> Because it silently loses tasks, which can violate business correctness.

### Q8. How can retry worsen overload?

> If rejected/failed tasks are retried immediately, retries add more work while the system is already saturated, creating a retry storm.

### Q9. Why use separate pools?

> To isolate workloads so one slow dependency or task category does not consume all workers needed by another category.

### Q10. How do you choose queue size?

> Based on latency targets, workload burstiness, processing time, memory constraints and downstream capacity; there is no universal queue-size formula.

---

# 26. Quick Revision

```text
Producer
   ↓
Bounded Queue
   ↓
Worker Pool
   ↓
Consumer

If queue full
   ↓
Max workers?
   ↓ no
Reject
   ↓
Policy
```

### Rejection Memory Trick

```text
Abort          → Exception
CallerRuns     → Caller executes
Discard        → Drop
DiscardOldest  → Drop oldest
```

### Backpressure Memory Trick

```text
Limit work
   +
Slow producer
   +
Reject safely
   +
Protect downstream
```

---

# 🎯 Interview Questions

1. What is queueing in a thread pool?
2. Why is queue capacity important?
3. Bounded vs unbounded queue?
4. Explain `ArrayBlockingQueue`.
5. Explain `LinkedBlockingQueue`.
6. What is `SynchronousQueue`?
7. What causes task rejection?
8. Explain all four rejection policies.
9. What is backpressure?
10. How does `CallerRunsPolicy` create producer slowdown?
11. Why is `DiscardPolicy` dangerous?
12. What is load shedding?
13. What is admission control?
14. How can retries cause cascading failure?
15. Why can a large queue increase latency?
16. How do you monitor executor saturation?
17. Why use separate thread pools?
18. How would you protect a slow downstream API?
19. How would you handle rejected payment-processing work?
20. How would you choose queue capacity?
21. Explain Core → Queue → Max → Reject.
22. Explain queueing and backpressure in 2 minutes.

---

# 🏆 2-Minute Interview Answer

> **"In a thread pool, the queue temporarily buffers tasks when the core workers are busy. A bounded queue is important because it puts an explicit limit on queued work. When the queue is full and the executor has already reached maximum worker count, the rejection policy determines what happens. Java provides Abort, CallerRuns, Discard and DiscardOldest policies. Backpressure means preventing producers from overwhelming consumers, for example by using bounded queues, producer slowdown, rate limiting, admission control or controlled rejection. I would also monitor queue depth, execution latency and rejection count. In production, I would size the pool and queue based on measured workload and downstream capacity rather than simply making the queue or thread count larger."**

---

# 💻 Practice Checklist

- [ ] Create a bounded `ArrayBlockingQueue`.
- [ ] Observe queue growth.
- [ ] Demonstrate Core → Queue → Max → Reject.
- [ ] Practice `AbortPolicy`.
- [ ] Practice `CallerRunsPolicy`.
- [ ] Practice `DiscardPolicy`.
- [ ] Practice `DiscardOldestPolicy`.
- [ ] Compare bounded and unbounded queues.
- [ ] Simulate a fast producer and slow consumer.
- [ ] Measure producer slowdown with `CallerRunsPolicy`.
- [ ] Handle `RejectedExecutionException` explicitly.
- [ ] Simulate overload with an external API.
- [ ] Design separate pools for workload isolation.
- [ ] Track queue depth and rejection count.
- [ ] Explain backpressure in under 2 minutes.

---

## Navigation

[← 8.8 — `ThreadPoolExecutor` Internals](../08-ThreadPoolExecutor-Internals/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.10 — Custom `ThreadFactory`**