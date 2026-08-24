# 8.35 — Fork/Join Framework

> **Goal:** Understand Java's Fork/Join Framework for divide-and-conquer parallelism, `RecursiveTask`, `RecursiveAction`, `ForkJoinPool`, work stealing, thresholds, and interview-level implementation.

---

## 1. Core Mental Model ⭐⭐⭐⭐⭐

```text
Large Problem
     ↓
split / fork
   ↙     ↘
small   small
task     task
   ↘     ↙
   join / combine
      ↓
   final result
```

Memory trick:

```text
Fork  = split work
Join  = combine/wait for result
```

The framework is designed especially for **recursive divide-and-conquer tasks**.

---

## 2. Main Classes ⭐⭐⭐⭐⭐

| Class | Purpose |
|---|---|
| `ForkJoinPool` | Executes Fork/Join tasks |
| `RecursiveTask<V>` | Recursive task that returns a result |
| `RecursiveAction` | Recursive task that returns no result |
| `ForkJoinTask<V>` | Base abstraction for Fork/Join tasks |
|

Most interview examples use:

```java
RecursiveTask<Integer>
```

for calculations and:

```java
RecursiveAction
```

for side-effecting work.

---

## 3. Why Fork/Join? ⭐⭐⭐⭐⭐

Suppose we need to sum a very large array.

Sequential:

```text
[1 2 3 4 5 6 7 8]
       ↓
   one thread
```

Fork/Join:

```text
             [1..8]
             /    \
          [1..4]  [5..8]
          /  \      /  \
       [1..2][3..4][5..6][7..8]
```

Small tasks can execute concurrently and then their results are combined.

Important:

> Fork/Join is not automatically faster for every problem. The problem should have enough independent work to justify parallelism and task-management overhead.

---

# 4. `RecursiveTask<V>` ⭐⭐⭐⭐⭐

Use when the recursive computation returns a value.

Skeleton:

```java
class MyTask extends RecursiveTask<Integer> {

    @Override
    protected Integer compute() {
        // split or calculate directly
    }
}
```

Typical pattern:

```java
if (problemIsSmall()) {
    return calculateDirectly();
}

MyTask left = new MyTask(...);
MyTask right = new MyTask(...);

left.fork();
Integer rightResult = right.compute();
Integer leftResult = left.join();

return leftResult + rightResult;
```

---

# 5. First Practice — Basic `RecursiveTask` ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class BasicRecursiveTaskDemo {

    static class SumTask extends RecursiveTask<Integer> {

        private final int[] array;
        private final int start;
        private final int end;

        SumTask(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Integer compute() {

            if (end - start <= 2) {
                int sum = 0;
                for (int i = start; i < end; i++) {
                    sum += array[i];
                }
                return sum;
            }

            int mid = (start + end) / 2;

            SumTask left = new SumTask(array, start, mid);
            SumTask right = new SumTask(array, mid, end);

            left.fork();

            int rightResult = right.compute();
            int leftResult = left.join();

            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {

        int[] numbers = {1, 2, 3, 4, 5, 6, 7, 8};

        ForkJoinPool pool = new ForkJoinPool();

        int result = pool.invoke(
                new SumTask(numbers, 0, numbers.length)
        );

        System.out.println("Sum = " + result);

        pool.shutdown();
    }
}
```

Expected output:

```text
Sum = 36
```

---

# 6. Understand `compute()` ⭐⭐⭐⭐⭐

The most important method is:

```java
protected Integer compute()
```

Inside it we decide:

```text
Is the task small enough?
      ↓
 YES → calculate directly
 NO  → split into subtasks
```

This is the **divide-and-conquer** pattern.

---

# 7. What Does `fork()` Do? ⭐⭐⭐⭐⭐

```java
left.fork();
```

It schedules the task for asynchronous execution in the Fork/Join pool.

It does not mean:

```text
new Thread()
```

for every recursive task.

Fork/Join uses worker threads managed by `ForkJoinPool`.

---

# 8. What Does `join()` Do? ⭐⭐⭐⭐⭐

```java
int leftResult = left.join();
```

It waits for the task's completion and obtains its result.

Important interview distinction:

```text
fork() → schedule task
join() → wait/get result
```

---

# 9. Why `right.compute()` Instead of `right.fork()`? ⭐⭐⭐⭐⭐

A common optimized pattern is:

```java
left.fork();
int rightResult = right.compute();
int leftResult = left.join();
```

Instead of:

```java
left.fork();
right.fork();
int leftResult = left.join();
int rightResult = right.join();
```

The first pattern lets the current worker execute one branch directly while the other branch is forked.

It can reduce unnecessary scheduling overhead and is a common Fork/Join idiom.

---

# 10. `ForkJoinPool` ⭐⭐⭐⭐⭐

Create a pool:

```java
ForkJoinPool pool = new ForkJoinPool();
```

Submit a task:

```java
int result = pool.invoke(task);
```

`invoke()` submits the task and waits for its completion, returning the result.

You can also use:

```java
pool.execute(task);
```

or:

```java
pool.submit(task);
```

which provide asynchronous submission semantics.

---

# 11. `RecursiveAction` ⭐⭐⭐⭐⭐

Use `RecursiveAction` when the task does not return a value.

```java
class PrintTask extends RecursiveAction {

    @Override
    protected void compute() {
        // work
    }
}
```

Example:

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class RecursiveActionDemo {

    static class PrintTask extends RecursiveAction {

        private final int start;
        private final int end;

        PrintTask(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        protected void compute() {

            if (end - start <= 2) {
                for (int i = start; i < end; i++) {
                    System.out.println(
                            Thread.currentThread().getName()
                                    + " -> " + i
                    );
                }
                return;
            }

            int mid = (start + end) / 2;

            PrintTask left = new PrintTask(start, mid);
            PrintTask right = new PrintTask(mid, end);

            invokeAll(left, right);
        }
    }

    public static void main(String[] args) {

        ForkJoinPool pool = new ForkJoinPool(4);

        pool.invoke(new PrintTask(0, 8));

        pool.shutdown();
    }
}
```

---

# 12. `invokeAll()` ⭐⭐⭐⭐⭐

Inside a Fork/Join task:

```java
invokeAll(left, right);
```

is a convenient way to fork multiple subtasks and wait for their completion.

For two subtasks, it is often cleaner than manually writing:

```java
left.fork();
right.fork();
left.join();
right.join();
```

---

# 13. Threshold — The Most Important Performance Concept ⭐⭐⭐⭐⭐

Do NOT recursively split until one element unless the problem is tiny.

Use a threshold:

```java
if (end - start <= THRESHOLD) {
    return calculateDirectly();
}
```

Why?

Every task has overhead:

```text
create task
schedule task
manage task
join task
```

If tasks become too small:

```text
parallelism benefit ↓
overhead ↑
```

So choose a practical threshold based on workload and benchmark it.

---

# 14. Threshold Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ThresholdSumDemo {

    static class SumTask extends RecursiveTask<Long> {

        private static final int THRESHOLD = 1_000;

        private final int[] numbers;
        private final int start;
        private final int end;

        SumTask(int[] numbers, int start, int end) {
            this.numbers = numbers;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {

            if (end - start <= THRESHOLD) {
                long sum = 0;
                for (int i = start; i < end; i++) {
                    sum += numbers[i];
                }
                return sum;
            }

            int mid = (start + end) >>> 1;

            SumTask left = new SumTask(numbers, start, mid);
            SumTask right = new SumTask(numbers, mid, end);

            left.fork();

            long rightResult = right.compute();
            long leftResult = left.join();

            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {

        int[] numbers = new int[100_000];

        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = i + 1;
        }

        ForkJoinPool pool = new ForkJoinPool();

        long sum = pool.invoke(
                new SumTask(numbers, 0, numbers.length)
        );

        System.out.println("Sum = " + sum);

        pool.shutdown();
    }
}
```

---

# 15. Work Stealing ⭐⭐⭐⭐⭐

Fork/Join's major feature is **work stealing**.

Conceptually:

```text
Worker-1 deque → tasks
Worker-2 deque → tasks
Worker-3 deque → tasks
Worker-4 deque → tasks
```

If a worker finishes its own work, it can steal available work from another worker's queue.

This helps keep workers busy when recursive tasks have uneven workloads.

Interview line:

> **ForkJoinPool uses work stealing so idle worker threads can take available tasks from other workers' queues, improving utilization for recursive parallel workloads.**

---

# 16. Work-Stealing Mental Model

```text
             ForkJoinPool
        ┌──────┬──────┬──────┐
        ↓      ↓      ↓      ↓
      W1     W2     W3     W4
     queue  queue  queue  queue
        ↑             ↑
        └── steal ────┘
```

The exact internal implementation details are JVM/JDK implementation concerns; focus on the work-stealing model for interviews.

---

# 17. `ForkJoinPool.commonPool()` ⭐⭐⭐⭐⭐

Java also provides a shared common pool:

```java
ForkJoinPool.commonPool()
```

Many async APIs use the common pool by default, depending on the API.

You can inspect parallelism:

```java
System.out.println(
        ForkJoinPool.commonPool().getParallelism()
);
```

Interview caution:

> Do not assume the common pool is always the right executor for arbitrary blocking application work.

---

# 18. Custom `ForkJoinPool` ⭐⭐⭐⭐⭐

```java
ForkJoinPool pool = new ForkJoinPool(4);
```

Here:

```text
parallelism = 4
```

This gives you a dedicated Fork/Join pool instead of relying on the common pool.

Use a custom pool when you need explicit isolation and the workload justifies it.

---

# 19. Complete Interview Example — Parallel Maximum ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ParallelMaxDemo {

    static class MaxTask extends RecursiveTask<Integer> {

        private static final int THRESHOLD = 10;

        private final int[] numbers;
        private final int start;
        private final int end;

        MaxTask(int[] numbers, int start, int end) {
            this.numbers = numbers;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Integer compute() {

            if (end - start <= THRESHOLD) {
                int max = Integer.MIN_VALUE;

                for (int i = start; i < end; i++) {
                    max = Math.max(max, numbers[i]);
                }

                return max;
            }

            int mid = (start + end) >>> 1;

            MaxTask left = new MaxTask(numbers, start, mid);
            MaxTask right = new MaxTask(numbers, mid, end);

            left.fork();

            int rightMax = right.compute();
            int leftMax = left.join();

            return Math.max(leftMax, rightMax);
        }
    }

    public static void main(String[] args) {

        int[] numbers = {
                10, 45, 7, 99, 23, 81, 56, 3,
                72, 18, 120, 34
        };

        ForkJoinPool pool = new ForkJoinPool(4);

        int max = pool.invoke(
                new MaxTask(numbers, 0, numbers.length)
        );

        System.out.println("Maximum = " + max);

        pool.shutdown();
    }
}
```

Expected:

```text
Maximum = 120
```

---

# 20. Complete Interview Example — Parallel Search ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ParallelSearchDemo {

    static class SearchTask extends RecursiveTask<Integer> {

        private static final int THRESHOLD = 100;

        private final int[] numbers;
        private final int start;
        private final int end;
        private final int target;

        SearchTask(int[] numbers, int start, int end, int target) {
            this.numbers = numbers;
            this.start = start;
            this.end = end;
            this.target = target;
        }

        @Override
        protected Integer compute() {

            if (end - start <= THRESHOLD) {
                for (int i = start; i < end; i++) {
                    if (numbers[i] == target) {
                        return i;
                    }
                }
                return -1;
            }

            int mid = (start + end) >>> 1;

            SearchTask left =
                    new SearchTask(numbers, start, mid, target);

            SearchTask right =
                    new SearchTask(numbers, mid, end, target);

            left.fork();

            int rightResult = right.compute();
            int leftResult = left.join();

            return leftResult != -1 ? leftResult : rightResult;
        }
    }

    public static void main(String[] args) {

        int[] numbers = new int[10_000];

        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = i;
        }

        int target = 9_876;

        ForkJoinPool pool = new ForkJoinPool();

        int index = pool.invoke(
                new SearchTask(
                        numbers,
                        0,
                        numbers.length,
                        target
                )
        );

        System.out.println("Index = " + index);

        pool.shutdown();
    }
}
```

---

# 21. Important Caveat: Parallel Search and Cancellation ⭐⭐⭐⭐

The previous example may continue processing another branch even after a match is found.

A production implementation may need:

```text
shared cancellation signal
short-circuit logic
careful task coordination
```

Do not claim that `join()` automatically cancels unrelated work.

---

# 22. Fork/Join vs `ExecutorService` ⭐⭐⭐⭐⭐

| Feature | Fork/Join | `ExecutorService` |
|---|---|---|
| Best fit | Recursive divide-and-conquer | General task execution |
| Main abstraction | `RecursiveTask` / `RecursiveAction` | `Runnable` / `Callable` |
| Pool | `ForkJoinPool` | Various executors |
| Work stealing | Yes | Not the defining feature |
| Recursive splitting | Natural | Manual |
| Result | `ForkJoinTask` result | `Future` |

Interview answer:

> **I prefer Fork/Join when the workload naturally decomposes recursively into smaller independent tasks. For general application task execution, ExecutorService is usually the more direct abstraction.**

---

# 23. Fork/Join vs Parallel Stream ⭐⭐⭐⭐⭐

Parallel streams can use Fork/Join infrastructure internally, but the programming model is different.

```java
numbers.parallelStream()
```

lets the stream framework manage splitting and execution.

Fork/Join gives explicit control over:

```text
splitting
threshold
recursive task structure
pool
combination
```

Use explicit Fork/Join when you need a custom recursive algorithm rather than simply parallelizing a stream pipeline.

---

# 24. Blocking Work Warning 🚨

Fork/Join is designed primarily for CPU-oriented parallel tasks.

Be cautious with long blocking operations:

```java
database.call();
httpClient.call();
file.read();
```

Blocking can occupy worker threads and reduce effective parallelism.

For blocking workloads, consider a suitable executor and workload isolation rather than blindly putting blocking operations into a Fork/Join pool.

---

# 25. Common Interview Trap — `fork()` Creates a New Thread ❌

Wrong:

> Every `fork()` creates a new thread.

Correct:

> `fork()` schedules a `ForkJoinTask` for execution in a `ForkJoinPool`; it does not create one new Java thread per task.

---

# 26. Common Interview Trap — Fork/Join Is Always Faster ❌

Wrong:

> Fork/Join always improves performance.

Correct:

> Fork/Join can improve CPU-bound divide-and-conquer workloads when there is enough independent work, but task overhead, contention, memory behavior and available CPU cores can make sequential execution faster for small workloads.

---

# 27. Common Interview Trap — `join()` Means Normal Thread Join ❌

`ForkJoinTask.join()` is related conceptually to waiting for completion, but it is a task-level operation integrated with Fork/Join execution.

Do not confuse it with:

```java
Thread.join()
```

---

# 28. Production Scenario — Large File/Data Processing 🏆

Potential workload:

```text
Large dataset
     ↓
split ranges
     ↓
process chunks
     ↓
combine partial results
```

Example use cases can include CPU-heavy:

```text
image transformation
large-array calculations
recursive tree processing
data aggregation
search/index calculations
```

The workload must be evaluated for CPU cost, task granularity, memory pressure and scalability before choosing Fork/Join.

---

# 29. `RecursiveTask` vs `RecursiveAction` ⭐⭐⭐⭐⭐

```text
Need result?
   ↓
 YES → RecursiveTask<V>
 NO  → RecursiveAction
```

Example:

```java
class SumTask extends RecursiveTask<Long>
```

returns:

```java
Long
```

Whereas:

```java
class PrintTask extends RecursiveAction
```

returns nothing.

---

# 30. Exception Handling ⭐⭐⭐⭐

Exceptions from a Fork/Join task can be observed when the task result is retrieved.

Example:

```java
try {
    pool.invoke(task);
} catch (RuntimeException e) {
    System.out.println("Task failed: " + e.getMessage());
}
```

For multiple recursive subtasks, define clearly how failures should affect the parent computation.

---

# 31. Cancellation ⭐⭐⭐⭐

Fork/Join tasks support cancellation through the `ForkJoinTask` API.

But cancellation is cooperative and application logic should be designed appropriately.

Do not assume cancellation instantly stops arbitrary computation already executing.

---

# 32. Performance Checklist ⭐⭐⭐⭐⭐

Before using Fork/Join, ask:

```text
1. Is the workload CPU-bound?
2. Can it be split into independent subtasks?
3. Is each subtask large enough?
4. Is the threshold sensible?
5. Is combining results cheap?
6. Is memory usage acceptable?
7. Is the common pool appropriate?
8. Do I need a custom pool?
9. Are there blocking calls?
10. Did I benchmark it?
```

---

# 33. 🏆 COMPLETE INTERVIEW PRACTICE CODE

Write this code **without looking** after studying the previous examples.

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ForkJoinInterviewPractice {

    static class SumTask extends RecursiveTask<Long> {

        private static final int THRESHOLD = 1_000;

        private final int[] numbers;
        private final int start;
        private final int end;

        SumTask(int[] numbers, int start, int end) {
            this.numbers = numbers;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {

            // 1. Base case
            if (end - start <= THRESHOLD) {

                long sum = 0;

                for (int i = start; i < end; i++) {
                    sum += numbers[i];
                }

                return sum;
            }

            // 2. Split
            int mid = (start + end) >>> 1;

            SumTask left =
                    new SumTask(numbers, start, mid);

            SumTask right =
                    new SumTask(numbers, mid, end);

            // 3. Fork one branch
            left.fork();

            // 4. Compute other branch directly
            long rightResult = right.compute();

            // 5. Join forked branch
            long leftResult = left.join();

            // 6. Combine
            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {

        // Input
        int[] numbers = new int[100_000];

        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = i + 1;
        }

        // Pool
        ForkJoinPool pool =
                new ForkJoinPool(4);

        try {

            // Execute
            long result = pool.invoke(
                    new SumTask(
                            numbers,
                            0,
                            numbers.length
                    )
            );

            System.out.println("Total = " + result);

        } finally {

            // Shutdown
            pool.shutdown();
        }
    }
}
```

### Practice from memory

You should be able to write these six steps:

```text
1. Extend RecursiveTask<V>
2. Define base case / threshold
3. Split problem
4. fork one task
5. compute + join
6. combine results
```

---

# 34. 2-Minute Interview Answer 🏆

> **"Fork/Join is a Java concurrency framework designed mainly for divide-and-conquer workloads. I use `RecursiveTask<V>` when a recursive computation returns a result and `RecursiveAction` when it does not. Inside `compute()`, I first check a threshold; if the task is small enough, I calculate directly. Otherwise I split it into subtasks, fork one task, compute the other task, join the forked task, and combine the results. `ForkJoinPool` executes these tasks and uses work stealing so idle workers can pick available work from other workers. A key performance consideration is choosing an appropriate threshold because creating too many tiny tasks adds overhead. Fork/Join is a good fit for CPU-bound recursive problems, but I would be careful with blocking I/O because it can occupy worker threads and reduce effective parallelism."**

---

# 35. 30-Second Hinglish Answer

> **"Fork/Join Framework divide-and-conquer problems ke liye use hota hai. Large task ko smaller subtasks mein split karte hain. `RecursiveTask` result return karta hai aur `RecursiveAction` result return nahi karta. `compute()` mein threshold check karte hain; small task direct calculate hota hai, otherwise split, `fork()`, doosre task ko `compute()`, phir `join()` karke result combine karte hain. `ForkJoinPool` work stealing use karta hai. Important hai ki task bahut small na ho, warna scheduling overhead benefit ko reduce kar sakta hai."**

---

# 36. Rapid-Fire Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is Fork/Join?

Divide-and-conquer parallelism framework.

### Q2. What is `ForkJoinPool`?

Pool designed to execute Fork/Join tasks and support work stealing.

### Q3. `RecursiveTask` vs `RecursiveAction`?

```text
RecursiveTask<V> → result
RecursiveAction → no result
```

### Q4. What does `fork()` do?

Schedules a task for asynchronous execution in the Fork/Join pool.

### Q5. What does `join()` do?

Waits for the task's completion and returns its result.

### Q6. What is work stealing?

Idle workers can take available tasks from other workers' queues.

### Q7. Why use a threshold?

To prevent overhead from excessive tiny tasks.

### Q8. Is Fork/Join always faster?

No.

### Q9. Is Fork/Join good for blocking I/O?

Generally be cautious; CPU-oriented work is the primary fit.

### Q10. `ForkJoinPool` vs `ExecutorService`?

Fork/Join is specialized for recursive divide-and-conquer; ExecutorService is general-purpose task execution.

### Q11. What is the common pool?

A shared `ForkJoinPool` available through `ForkJoinPool.commonPool()`.

### Q12. Why fork one branch and compute the other?

It can reduce unnecessary scheduling overhead and lets the current worker continue useful work.

---

# 37. Quick Revision 🧠

```text
Fork/Join
   ↓
Divide & Conquer
   ↓
RecursiveTask / RecursiveAction
   ↓
compute()
   ↓
threshold?
 ┌───────┴───────┐
 YES             NO
  ↓               ↓
calculate       split
                ↓
             fork one
                ↓
            compute other
                ↓
               join
                ↓
             combine
```

### Golden Rules

```text
Fork = schedule
Join = wait/get
Threshold = control task granularity
ForkJoinPool = execution pool
Work stealing = worker utilization
RecursiveTask = result
RecursiveAction = no result
```

---

# 38. 💻 Practice Checklist

- [ ] Write a `RecursiveTask<Integer>`
- [ ] Write a `RecursiveTask<Long>`
- [ ] Write a `RecursiveAction`
- [ ] Implement array sum
- [ ] Implement maximum search
- [ ] Implement parallel search
- [ ] Explain `compute()`
- [ ] Explain `fork()`
- [ ] Explain `join()`
- [ ] Explain work stealing
- [ ] Explain threshold
- [ ] Create custom `ForkJoinPool`
- [ ] Explain common pool
- [ ] Compare Fork/Join with ExecutorService
- [ ] Compare Fork/Join with parallel streams
- [ ] Explain blocking-work risk
- [ ] Explain why Fork/Join is not always faster
- [ ] Write complete interview code without looking
- [ ] Give the 2-minute interview answer
- [ ] Give the 30-second Hinglish answer

---

## Navigation

[← 8.34 — `allOf()` / `anyOf()`](../34-allOf-and-anyOf/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.36 — Recursive Tasks & Work Stealing**