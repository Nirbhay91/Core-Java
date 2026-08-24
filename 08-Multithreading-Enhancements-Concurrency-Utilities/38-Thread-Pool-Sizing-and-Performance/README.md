# 8.38 — Thread Pool Sizing & Performance

> **Goal:** Learn how to choose thread-pool sizes, reason about CPU-bound vs I/O-bound workloads, queue capacity, throughput, latency, saturation, and production tuning.

## 1. Core Mental Model ⭐⭐⭐⭐⭐

```text
Tasks
  ↓
Thread Pool
  ├── Worker Threads
  ├── Queue
  └── Rejection Policy
```

Thread-pool sizing is a **workload and resource problem**, not a fixed-number problem.

---

# 2. CPU-Bound vs I/O-Bound ⭐⭐⭐⭐⭐

### CPU-bound

Examples:

```text
Sorting
Encryption
Compression
Complex calculations
Image processing
```

Threads spend most of their time using CPU.

A useful starting heuristic is:

```text
threads ≈ number of available processors
```

but the final number must be measured and tuned.

### I/O-bound

Examples:

```text
HTTP calls
Database calls
File I/O
Remote service calls
```

Threads may spend significant time waiting.

A rough model sometimes used for initial estimation is:

```text
N_threads ≈ N_cpu × U_cpu × (1 + W/C)
```

Where:

```text
N_cpu = available processors
U_cpu = desired CPU utilization
W     = wait time
C     = compute time
```

This is a **starting model, not a universal formula**.

---

# 3. Why `availableProcessors()` Matters ⭐⭐⭐⭐⭐

```java
int cores = Runtime.getRuntime().availableProcessors();

System.out.println("Available processors = " + cores);
```

Use it as an input to sizing decisions, not as an automatic answer.

Containerized applications can have CPU limits that make blindly assuming host CPU count dangerous; validate the runtime environment and benchmark the real deployment.

---

# 4. Basic Fixed Thread Pool ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedPoolDemo {

    public static void main(String[] args) {

        int cores = Runtime.getRuntime().availableProcessors();

        ExecutorService executor =
                Executors.newFixedThreadPool(cores);

        try {
            for (int i = 1; i <= 10; i++) {
                int taskId = i;

                executor.submit(() -> {
                    System.out.println(
                            Thread.currentThread().getName()
                                    + " executing task " + taskId
                    );
                });
            }
        } finally {
            executor.shutdown();
        }
    }
}
```

Important: `cores` is only a reasonable CPU-bound starting point, not a universal production configuration.

---

# 5. Thread Count Too Low ⭐⭐⭐⭐⭐

Suppose:

```text
100 tasks
2 worker threads
```

Then:

```text
100 tasks
   ↓
2 workers
   ↓
large queue / long wait
   ↓
higher latency
```

Potential symptoms:

- Queue grows
- Task waiting time increases
- Throughput may be unnecessarily limited
- Request latency can increase

But increasing threads blindly is not the solution.

---

# 6. Thread Count Too High 🚨⭐⭐⭐⭐⭐

Suppose:

```text
CPU cores = 8
Threads = 500
```

For CPU-bound work, this can create:

```text
Context switching
Scheduling overhead
Cache disruption
CPU contention
Memory overhead
```

More threads ≠ more throughput.

---

# 7. Queue Capacity Matters ⭐⭐⭐⭐⭐

Thread pool behavior is roughly:

```text
New task
   ↓
Can a worker take it?
   ↓
Can a new worker be created?
   ↓
Can it enter the queue?
   ↓
Otherwise → reject
```

Queue capacity is therefore part of your concurrency design.

---

# 8. Bounded Queue Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class BoundedQueueDemo {

    public static void main(String[] args) {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                4,
                8,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        try {
            for (int i = 1; i <= 1_000; i++) {
                int taskId = i;

                executor.execute(() -> {
                    System.out.println("Task " + taskId);
                });
            }
        } finally {
            executor.shutdown();
        }
    }
}
```

This demonstrates explicit control over:

```text
corePoolSize
maximumPoolSize
keepAliveTime
queue capacity
rejection policy
```

---

# 9. `corePoolSize` vs `maximumPoolSize` ⭐⭐⭐⭐⭐

```java
new ThreadPoolExecutor(
        4,      // core
        8,      // maximum
        30,
        TimeUnit.SECONDS,
        queue
);
```

Conceptually:

```text
Tasks arrive
   ↓
Use core workers
   ↓
Queue according to executor policy
   ↓
If queue cannot accept and max not reached
   ↓
Create workers up to maximum
   ↓
If saturated → rejection policy
```

A subtle but important interview point:

> With an unbounded queue such as `LinkedBlockingQueue` used without a capacity, a `ThreadPoolExecutor` generally does not need to grow beyond `corePoolSize` for queued tasks, so `maximumPoolSize` may effectively have little practical impact.

---

# 10. Rejection as Backpressure ⭐⭐⭐⭐⭐

When the pool and queue are saturated:

```text
Workers full
   +
Queue full
   ↓
Task rejected
```

Policies include:

```java
AbortPolicy
CallerRunsPolicy
DiscardPolicy
DiscardOldestPolicy
```

### `CallerRunsPolicy`

The submitting thread executes the task.

This can naturally slow the producer and act as a form of backpressure.

---

# 11. Why Unbounded Queues Can Be Dangerous 🚨⭐⭐⭐⭐⭐

Example:

```java
new LinkedBlockingQueue<>();
```

If producers continuously submit faster than workers can process:

```text
Producer rate > Consumer rate
          ↓
Queue keeps growing
          ↓
Memory pressure
          ↓
Latency increases
          ↓
Potential instability
```

An unbounded queue removes rejection as a normal overload signal, which can hide saturation.

---

# 12. Queueing Delay vs Processing Time ⭐⭐⭐⭐⭐

Total task latency can be thought of as:

```text
Latency
  = queue waiting time
  + execution time
  + downstream waiting
```

A service can have fast task execution but poor end-to-end latency because tasks spend most of their time waiting in the queue.

---

# 13. Little's Law ⭐⭐⭐⭐

A useful performance relationship is:

```text
L = λ × W
```

Where:

```text
L = average number of items in the system
λ = throughput / arrival rate
W = average time in the system
```

This helps reason about queue depth, throughput and latency.

Example:

```text
100 requests/sec
×
0.5 sec average latency
=
50 requests in the system
```

It is a steady-state relationship, not a magic sizing formula.

---

# 14. CPU Utilization and Pool Size ⭐⭐⭐⭐⭐

For CPU-bound work:

```text
Too few threads
    ↓
CPU may be underutilized
```

Too many threads:

```text
CPU contention
    ↓
Context switching
    ↓
Less useful work
```

Target:

```text
Enough parallelism
without excessive contention
```

---

# 15. I/O-Bound Pool Sizing ⭐⭐⭐⭐⭐

If workers spend significant time waiting:

```text
Thread
  ↓
HTTP request
  ↓
WAIT
  ↓
response
  ↓
CPU work
```

More concurrency can improve utilization, but only within downstream limits.

Consider:

```text
DB connection pool
HTTP connection pool
Remote service rate limits
Memory
CPU
Thread count
Timeouts
```

---

# 16. Downstream Capacity Is a Hard Constraint 🚨⭐⭐⭐⭐⭐

Example:

```text
Application pool = 100 threads
DB connection pool = 10
```

If every task needs a DB connection:

```text
100 threads
   ↓
10 DB connections
   ↓
90 threads wait
```

Increasing application threads may make the system worse.

Interview line:

> **The thread pool cannot be tuned independently from downstream resource pools.**

---

# 17. Connection Pool + Thread Pool Relationship ⭐⭐⭐⭐⭐

Think:

```text
Request threads
      ↓
DB connection pool
      ↓
Database
```

If:

```text
Thread pool >> DB pool
```

you may create excessive waiting.

If:

```text
Thread pool << DB pool
```

you may underutilize available DB capacity.

The correct value depends on workload, query latency, DB capacity and application behavior.

---

# 18. Timeout Is Part of Pool Design ⭐⭐⭐⭐⭐

Bad:

```java
future.get();
```

without considering how long the task can wait.

Better:

```java
future.get(2, TimeUnit.SECONDS);
```

A timeout prevents indefinite waiting at that call site.

For production systems also consider:

```text
HTTP timeout
DB timeout
queue timeout
future timeout
circuit breaker
```

---

# 19. Keep-Alive Time ⭐⭐⭐⭐

```java
new ThreadPoolExecutor(
        4,
        20,
        30,
        TimeUnit.SECONDS,
        queue
);
```

`keepAliveTime` controls how long eligible excess idle workers are retained before termination.

With the default executor configuration, this primarily applies to workers beyond the core size; `allowCoreThreadTimeOut(true)` changes that behavior for core threads.

---

# 20. Allow Core Thread Timeout ⭐⭐⭐⭐

```java
executor.allowCoreThreadTimeOut(true);
```

Useful when you want idle core workers to eventually terminate, depending on workload characteristics.

Trade-off:

```text
Less idle resource usage
        vs
Thread recreation/warm-up cost
```

---

# 21. Measuring Pool State ⭐⭐⭐⭐⭐

`ThreadPoolExecutor` exposes useful runtime metrics:

```java
ThreadPoolExecutor executor = ...;

System.out.println("Pool size = " + executor.getPoolSize());
System.out.println("Active = " + executor.getActiveCount());
System.out.println("Queued = " + executor.getQueue().size());
System.out.println("Completed = " + executor.getCompletedTaskCount());
System.out.println("Largest = " + executor.getLargestPoolSize());
```

These are useful for observability and diagnosis.

---

# 22. Monitoring Example 🏆

```java
import java.util.concurrent.*;

public class PoolMonitoringDemo {

    public static void main(String[] args) throws Exception {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10)
        );

        try {
            for (int i = 1; i <= 20; i++) {
                int id = i;

                executor.execute(() -> {
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }

                    System.out.println("Task " + id + " completed");
                });
            }

            while (!executor.isTerminated()) {
                System.out.printf(
                        "pool=%d active=%d queue=%d completed=%d%n",
                        executor.getPoolSize(),
                        executor.getActiveCount(),
                        executor.getQueue().size(),
                        executor.getCompletedTaskCount()
                );

                Thread.sleep(200);

                if (executor.isShutdown() && executor.getQueue().isEmpty()) {
                    break;
                }
            }
        } finally {
            executor.shutdown();
        }
    }
}
```

---

# 23. Warm-Up and Cold Start ⭐⭐⭐⭐

A pool may experience startup effects:

```text
No workers
   ↓
Task arrives
   ↓
Worker creation
   ↓
Task starts
```

For latency-sensitive workloads, consider whether prestarting core threads is appropriate:

```java
executor.prestartAllCoreThreads();
```

This is a tuning option, not a default requirement.

---

# 24. Thread Count Is Not the Only Bottleneck ⭐⭐⭐⭐⭐

A slow service can be limited by:

```text
CPU
Memory
GC
Database
Network
Locks
Connection pools
Queue capacity
External APIs
Serialization
Disk
```

Increasing thread count only addresses one part of the system.

---

# 25. Queue Size Is a Latency vs Absorption Trade-Off ⭐⭐⭐⭐⭐

Large queue:

```text
Absorbs bursts
        ↓
But may increase waiting time
```

Small queue:

```text
Less waiting
        ↓
Earlier backpressure/rejection
```

There is no universally correct queue size.

---

# 26. Bursty Traffic Scenario 🏆

Imagine:

```text
Normal arrival = 50/sec
Burst = 500/sec
Processing capacity = 100/sec
```

A queue can temporarily absorb the burst.

But if:

```text
arrival > processing capacity
for a sustained period
```

then:

```text
queue grows
   ↓
latency grows
   ↓
queue fills
   ↓
backpressure/rejection
```

A queue cannot permanently compensate for insufficient processing capacity.

---

# 27. CallerRunsPolicy as Backpressure ⭐⭐⭐⭐⭐

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        2,
        4,
        10,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(2),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

When saturated:

```text
Executor cannot accept
        ↓
Caller executes task
        ↓
Caller becomes busy
        ↓
Submission naturally slows
```

This can be useful, but only if making the caller execute that work is safe and acceptable.

---

# 28. CPU-Bound Example 🏆

```java
int cores = Runtime.getRuntime().availableProcessors();

ExecutorService executor =
        Executors.newFixedThreadPool(cores);
```

Starting point for CPU-bound work.

Then benchmark:

```text
cores - 1
cores
cores + 1
2 × cores
```

and observe throughput, CPU utilization, latency and context switching rather than assuming one formula is universally optimal.

---

# 29. I/O-Bound Example 🏆

Suppose:

```text
CPU = 8 cores
Task:
  10 ms CPU
  90 ms waiting
```

The task spends most of its time waiting.

A pool larger than 8 may improve throughput, subject to:

```text
Downstream capacity
Memory
Network
Timeouts
Connection pools
Remote rate limits
```

Do not simply multiply threads until performance improves; measure the saturation point.

---

# 30. Complete Interview Practice — Bounded Production Pool 🏆⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ProductionThreadPoolDemo {

    public static void main(String[] args) {

        int cores = Runtime.getRuntime().availableProcessors();

        int corePoolSize = cores;
        int maxPoolSize = cores * 2;
        int queueCapacity = 100;

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                corePoolSize,
                maxPoolSize,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(queueCapacity),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        try {
            for (int i = 1; i <= 1_000; i++) {
                int taskId = i;

                executor.execute(() -> process(taskId));
            }
        } finally {
            executor.shutdown();
        }
    }

    private static void process(int taskId) {
        System.out.println(
                Thread.currentThread().getName()
                        + " processing " + taskId
        );
    }
}
```

### Interview explanation

```text
corePoolSize
    ↓
normal worker capacity

maxPoolSize
    ↓
burst worker capacity

bounded queue
    ↓
prevents unlimited buffering

CallerRunsPolicy
    ↓
backpressure when saturated
```

Do **not** claim that `cores * 2` is universally correct. It is only an example configuration to discuss and benchmark.

---

# 31. Complete Interview Practice — Rejection Demo 🏆

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class RejectionDemo {

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
            for (int i = 1; i <= 5; i++) {
                int id = i;

                try {
                    executor.execute(() -> {
                        try {
                            Thread.sleep(1_000);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }

                        System.out.println("Task " + id);
                    });
                } catch (RejectedExecutionException e) {
                    System.out.println("Rejected task = " + id);
                }
            }
        } finally {
            executor.shutdown();
        }
    }
}
```

This is excellent for explaining:

```text
1 worker
1 queued task
remaining tasks
    ↓
rejected
```

---

# 32. Common Mistake — `newFixedThreadPool()` Everywhere 🚨

```java
Executors.newFixedThreadPool(100);
```

This may be inappropriate if:

```text
CPU = 4
```

and tasks are CPU-bound.

The correct size depends on workload and system constraints.

---

# 33. Common Mistake — Huge Queue 🚨

```java
new LinkedBlockingQueue<>();
```

combined with a high request rate can hide overload.

Symptoms:

```text
No immediate rejection
      ↓
Queue grows
      ↓
Requests become old
      ↓
Latency explodes
```

---

# 34. Common Mistake — Huge Maximum Pool 🚨

```java
new ThreadPoolExecutor(
        10,
        10_000,
        ...
);
```

This does not guarantee high throughput.

Potential consequences:

```text
Thread explosion
Memory pressure
Context switching
Downstream overload
```

---

# 35. Common Mistake — Pool per Request 🚨

Bad:

```java
public void handleRequest() {
    ExecutorService executor =
            Executors.newFixedThreadPool(10);

    executor.submit(this::doWork);
}
```

This can create excessive executors and threads.

Prefer appropriately managed/shared executors with a clear lifecycle.

---

# 36. Separate Pools for Different Workloads ⭐⭐⭐⭐⭐

Sometimes:

```text
CPU Pool
   ↓
CPU-intensive work

I/O Pool
   ↓
blocking I/O work
```

This can isolate workloads and prevent one class of work from monopolizing resources.

But separate pools also consume more resources and increase operational complexity, so use them where workload isolation provides value.

---

# 37. Thread Pool Sizing Interview Scenario 🏆⭐⭐⭐⭐⭐

### Question

> Your service has 8 CPU cores. What thread-pool size will you choose?

### Strong answer

> **"I wouldn't give one fixed number without knowing the workload. For CPU-bound work, I would start around the processor count and benchmark. For blocking I/O, I may need more concurrency because workers spend time waiting, but I would size it against downstream limits such as database connections and remote-service capacity. I would also choose a bounded queue and an overload strategy, then monitor queue depth, active threads, throughput, latency, CPU utilization and rejection rate."**

---

# 38. Thread Pool Sizing Interview Scenario — Latency ⭐⭐⭐⭐⭐

### Question

> CPU is only 30%, but API latency is very high. Should you increase threads?

### Answer

> **"Not automatically. I would first check queue wait time, downstream latency, connection-pool saturation, lock contention and whether the workers are blocked. Low CPU can mean the application is waiting on another resource. Increasing threads may help an I/O-bound bottleneck, but it can also amplify downstream pressure."**

---

# 39. Thread Pool Sizing Interview Scenario — Queue Growing ⭐⭐⭐⭐⭐

### Question

> Queue size keeps increasing. What does that tell you?

### Answer

> **"The arrival rate is exceeding the pool's effective processing capacity over that period. I would check task execution time, active workers, CPU, downstream latency, queue capacity, rejection rate and whether the workload is blocked. I would not simply increase the queue because that may convert overload into higher latency and memory pressure."**

---

# 40. Metrics to Monitor ⭐⭐⭐⭐⭐

At minimum:

```text
Active thread count
Pool size
Queue depth
Queue wait time
Task execution time
Completed task count
Rejected task count
CPU utilization
Memory/GC
Downstream latency
Connection-pool usage
Request latency
Throughput
Error/timeout rate
```

These metrics help determine whether the bottleneck is:

```text
CPU
Queue
Threads
Database
Network
External service
Locks
Memory
```

---

# 41. Performance Tuning Process 🧠⭐⭐⭐⭐⭐

Use this sequence:

```text
1. Measure baseline
       ↓
2. Identify bottleneck
       ↓
3. Form hypothesis
       ↓
4. Change one important variable
       ↓
5. Load test
       ↓
6. Compare throughput + latency
       ↓
7. Check resource utilization
       ↓
8. Validate under realistic traffic
```

Do not tune based on intuition alone.

---

# 42. Throughput vs Latency ⭐⭐⭐⭐⭐

A pool change can:

```text
increase throughput
BUT
increase latency
```

or:

```text
reduce latency
BUT
reduce throughput
```

Always define the business goal and observe both.

---

# 43. Backpressure Mental Model 🧠

```text
Producer
   ↓
Queue
   ↓
Workers
   ↓
Downstream
```

If downstream slows:

```text
workers wait
   ↓
queue grows
   ↓
backpressure
   ↓
reject / slow producers
```

A good design makes overload behavior explicit instead of allowing unlimited work to accumulate.

---

# 44. 🏆 Complete Interview Code — Monitoring + Backpressure

Write this from memory:

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class InterviewThreadPool {

    public static void main(String[] args) throws InterruptedException {

        int cores = Runtime.getRuntime().availableProcessors();

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                cores,
                cores * 2,
                30,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(50),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        try {
            for (int i = 1; i <= 500; i++) {
                int taskId = i;

                executor.execute(() -> {
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });

                if (i % 25 == 0) {
                    System.out.printf(
                            "submitted=%d pool=%d active=%d queue=%d completed=%d%n",
                            i,
                            executor.getPoolSize(),
                            executor.getActiveCount(),
                            executor.getQueue().size(),
                            executor.getCompletedTaskCount()
                    );
                }
            }
        } finally {
            executor.shutdown();
            executor.awaitTermination(1, TimeUnit.MINUTES);
        }
    }
}
```

### Explain these 5 things in interview:

```text
1. Why bounded queue?
2. Why CallerRunsPolicy?
3. Why core vs maximum?
4. What happens when queue is full?
5. Which metrics would you monitor?
```

---

# 45. 2-Minute Interview Answer 🏆

> **"Thread-pool sizing depends on workload. For CPU-bound tasks, I normally start near the number of available processors and benchmark. For I/O-bound tasks, more threads may be useful because workers spend time waiting, but the pool must be constrained by downstream resources such as database connections, HTTP connection pools and remote-service capacity. I also care about queue capacity because an unbounded queue can hide overload and increase latency indefinitely. In a production `ThreadPoolExecutor`, I would consider core pool size, maximum pool size, queue capacity, keep-alive time and rejection policy together. I would monitor active threads, queue depth, queue wait time, execution time, throughput, latency, CPU and rejection rate. I would tune using realistic load tests rather than relying on a fixed formula."**

---

# 46. 30-Second Hinglish Answer

> **"Thread pool ka size workload pe depend karta hai. CPU-bound task mein processors ke around se start karke benchmark karunga. I/O-bound task mein threads zyada ho sakte hain kyunki threads waiting mein rehte hain, but DB connection pool ya downstream API capacity ko bhi consider karna padega. Queue bounded rakhna useful hai taaki unlimited tasks memory mein accumulate na hon. Production mein active threads, queue size, latency, throughput, CPU aur rejection rate monitor karke tuning karunga."**

---

# 47. Rapid-Fire Interview Questions ⭐⭐⭐⭐⭐

### Q1. How do you size a CPU-bound pool?

Start around available processors and benchmark.

### Q2. How do you size an I/O-bound pool?

Use workload wait/compute characteristics plus downstream resource limits and benchmark.

### Q3. Is there one perfect formula?

No.

### Q4. Why can too many threads hurt?

Context switching, contention, memory overhead and downstream overload.

### Q5. Why bounded queue?

Explicit overload control and backpressure.

### Q6. Why can unbounded queue be dangerous?

It can hide overload, increase latency and consume memory.

### Q7. What is `CallerRunsPolicy`?

The submitting caller executes the rejected task.

### Q8. What happens when queue is full?

If the pool can still grow to maximum, it may create another worker; otherwise the rejection policy is invoked.

### Q9. Does maximum pool size always matter?

No. With an unbounded work queue, the pool generally won't grow beyond core size for queued tasks.

### Q10. What is queueing latency?

Time a task waits before execution.

### Q11. Why monitor queue depth?

It indicates pressure and potential saturation.

### Q12. Should thread pool size equal DB connection pool size?

Not necessarily; it depends on how tasks use the DB and other resources.

### Q13. CPU low + latency high — increase threads?

Not automatically; identify the actual bottleneck first.

### Q14. Can a larger queue improve throughput?

It may absorb bursts, but it does not increase sustainable processing capacity.

### Q15. How do you tune a pool?

Measure → identify bottleneck → change → load test → compare → monitor.

---

# 48. Practice Challenges 🔥

### Challenge 1
Create a fixed pool based on `availableProcessors()` and run CPU-heavy calculations.

### Challenge 2
Create a bounded `ThreadPoolExecutor` with:

```text
core = 2
max = 4
queue = 5
CallerRunsPolicy
```

Submit 30 slow tasks and observe queue/pool behavior.

### Challenge 3
Replace `CallerRunsPolicy` with `AbortPolicy` and observe rejection.

### Challenge 4
Create an unbounded queue and explain why `maximumPoolSize` behaves differently.

### Challenge 5
Print:

```text
pool size
active count
queue size
completed count
largest pool size
```

while tasks are running.

### Challenge 6
Create two pools:

```text
CPU pool
I/O pool
```

Explain why workload isolation may help.

### Challenge 7
Simulate a downstream bottleneck where 20 application threads compete for 5 connections. Explain why increasing application threads does not necessarily help.

---

# 49. Quick Revision 🧠

```text
CPU-bound
   ↓
start near CPU count
   ↓
benchmark
```

```text
I/O-bound
   ↓
more concurrency may help
   ↓
but respect downstream capacity
```

```text
Too few threads
   ↓
underutilization / queueing

Too many threads
   ↓
contention / context switching / overload
```

```text
Large queue
   ↓
absorbs bursts
   ↓
but may increase latency
```

```text
Bounded queue
   ↓
explicit saturation
   ↓
rejection/backpressure
```

### Golden Rules

```text
No universal thread count
Measure before tuning
CPU-bound ≠ I/O-bound
Queue size matters
Downstream capacity matters
More threads ≠ more throughput
More queue ≠ more capacity
Monitor queue wait + execution time
Use realistic load tests
Treat rejection as a design decision
```

---

# 50. Final Interview Checklist

- [ ] CPU-bound vs I/O-bound
- [ ] `availableProcessors()`
- [ ] Core pool size
- [ ] Maximum pool size
- [ ] Queue capacity
- [ ] Keep-alive time
- [ ] Rejection policies
- [ ] Backpressure
- [ ] Bounded vs unbounded queue
- [ ] Queueing latency
- [ ] Little's Law
- [ ] Downstream capacity
- [ ] DB connection pool relationship
- [ ] Timeout considerations
- [ ] Pool monitoring metrics
- [ ] CPU utilization
- [ ] Throughput vs latency
- [ ] Burst traffic
- [ ] Production pool design
- [ ] Complete interview code
- [ ] 2-minute interview answer
- [ ] 30-second Hinglish answer
- [ ] Rapid-fire questions

---

## Navigation

[← 8.37 — Parallel Streams & Concurrency Risks](../37-Parallel-Streams-and-Concurrency-Risks/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.39 — Graceful Shutdown & Production Patterns**