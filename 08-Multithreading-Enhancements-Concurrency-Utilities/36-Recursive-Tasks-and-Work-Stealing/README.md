# 8.36 — Recursive Tasks & Work Stealing

> **Goal:** Master recursive Fork/Join task design and understand how work stealing keeps Fork/Join workers busy.

## 1. Mental Model ⭐⭐⭐⭐⭐

```text
Large task
   ↓
split recursively
   ↓
small independent tasks
   ↓
fork / compute
   ↓
join
   ↓
combine
```

**Work stealing:** an idle Fork/Join worker can take available tasks from another worker's deque, helping balance uneven recursive workloads.

---

## 2. Recursive Task Pattern ⭐⭐⭐⭐⭐

```java
class MyTask extends RecursiveTask<Long> {
    @Override
    protected Long compute() {
        if (smallEnough()) {
            return directCalculation();
        }

        MyTask left = new MyTask(...);
        MyTask right = new MyTask(...);

        left.fork();
        long rightResult = right.compute();
        long leftResult = left.join();

        return leftResult + rightResult;
    }
}
```

Remember:

```text
Base case
   ↓
Split
   ↓
Fork one
   ↓
Compute one
   ↓
Join one
   ↓
Combine
```

---

# 3. Recursive Sum — Core Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class RecursiveSumDemo {

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

        ForkJoinPool pool = new ForkJoinPool(4);

        try {
            long result = pool.invoke(
                    new SumTask(numbers, 0, numbers.length)
            );

            System.out.println("Sum = " + result);
        } finally {
            pool.shutdown();
        }
    }
}
```

---

# 4. Why Fork One and Compute One? ⭐⭐⭐⭐⭐

Preferred common pattern:

```java
left.fork();
long rightResult = right.compute();
long leftResult = left.join();
```

The current worker immediately works on one branch while the other is made available to the pool. This can reduce unnecessary scheduling overhead compared with blindly forking both branches.

Do not claim this is an absolute performance rule for every algorithm; benchmark real workloads.

---

# 5. `invokeAll()` Alternative ⭐⭐⭐⭐

For multiple subtasks:

```java
invokeAll(left, right);
```

is a convenient Fork/Join operation that schedules the subtasks and waits for their completion.

Example:

```java
static class PrintTask extends RecursiveAction {

    private final int start;
    private final int end;

    PrintTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if (end - start <= 10) {
            for (int i = start; i < end; i++) {
                System.out.println(i);
            }
            return;
        }

        int mid = (start + end) >>> 1;

        PrintTask left = new PrintTask(start, mid);
        PrintTask right = new PrintTask(mid, end);

        invokeAll(left, right);
    }
}
```

---

# 6. Work Stealing ⭐⭐⭐⭐⭐

Conceptually, each Fork/Join worker has a deque of tasks:

```text
Worker-1 → [A][B][C]
Worker-2 → [D]
Worker-3 → []
Worker-4 → [E][F]
```

If Worker-3 becomes idle, it may steal available work from another worker.

```text
Worker-3
   ↓ steal
Worker-1's available task
```

The goal is better worker utilization when recursive branches have different execution times.

Interview line:

> **ForkJoinPool uses work stealing so idle workers can acquire available tasks from other workers, which helps balance recursive workloads.**

---

# 7. Work Stealing Is NOT Global FIFO ⭐⭐⭐⭐

Do not describe Fork/Join as a simple global FIFO queue.

The implementation is based on per-worker work queues/deques and stealing behavior. Exact internal scheduling details are implementation-specific, so interview answers should focus on the work-stealing model rather than claiming a simplistic queue order.

---

# 8. Uneven Workload Practice ⭐⭐⭐⭐⭐

This example helps visualize why work stealing matters.

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class UnevenWorkDemo {

    static class WorkTask extends RecursiveAction {

        private final int start;
        private final int end;

        WorkTask(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        protected void compute() {

            if (end - start <= 5) {
                simulateWork(start, end);
                return;
            }

            int mid = (start + end) >>> 1;

            WorkTask left = new WorkTask(start, mid);
            WorkTask right = new WorkTask(mid, end);

            invokeAll(left, right);
        }

        private void simulateWork(int start, int end) {
            for (int i = start; i < end; i++) {
                try {
                    Thread.sleep(i % 5 == 0 ? 50 : 5);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }

                System.out.println(
                        Thread.currentThread().getName()
                                + " processed " + i
                );
            }
        }
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool(4);

        try {
            pool.invoke(new WorkTask(0, 40));
        } finally {
            pool.shutdown();
        }
    }
}
```

**Interview caution:** this artificial example uses `sleep()` only to make uneven task duration visible. Fork/Join is primarily intended for computational work; blocking workloads need careful design.

---

# 9. Threshold and Recursive Granularity ⭐⭐⭐⭐⭐

Bad:

```java
if (end - start <= 1) {
    // direct work
}
```

for a very large array may create excessive task overhead.

Better:

```java
private static final int THRESHOLD = 1_000;
```

Then:

```java
if (end - start <= THRESHOLD) {
    return directCalculation();
}
```

There is no universal best threshold. It depends on:

```text
CPU
work per element
memory behavior
task overhead
pool parallelism
workload shape
```

Benchmark rather than guessing.

---

# 10. Recursive Tree Processing ⭐⭐⭐⭐⭐

Fork/Join is especially natural for recursive structures such as trees.

```text
             root
           /      \
         left     right
        /  \      /  \
      ...  ...  ...  ...
```

Each subtree can become an independent task.

---

# 11. Tree Size Example — Interview Practice ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class TreeSizeDemo {

    static class Node {
        int value;
        Node left;
        Node right;

        Node(int value) {
            this.value = value;
        }
    }

    static class SizeTask extends RecursiveTask<Integer> {

        private final Node node;

        SizeTask(Node node) {
            this.node = node;
        }

        @Override
        protected Integer compute() {

            if (node == null) {
                return 0;
            }

            SizeTask leftTask = new SizeTask(node.left);
            SizeTask rightTask = new SizeTask(node.right);

            leftTask.fork();

            int rightSize = rightTask.compute();
            int leftSize = leftTask.join();

            return 1 + leftSize + rightSize;
        }
    }

    public static void main(String[] args) {

        Node root = new Node(1);
        root.left = new Node(2);
        root.right = new Node(3);
        root.left.left = new Node(4);
        root.left.right = new Node(5);
        root.right.right = new Node(6);

        ForkJoinPool pool = new ForkJoinPool(4);

        try {
            int size = pool.invoke(new SizeTask(root));
            System.out.println("Tree size = " + size);
        } finally {
            pool.shutdown();
        }
    }
}
```

Expected:

```text
Tree size = 6
```

---

# 12. `RecursiveTask` vs `RecursiveAction` ⭐⭐⭐⭐⭐

```text
Need return value?
       ↓
     YES
       ↓
RecursiveTask<V>

Need only perform work?
       ↓
      NO
       ↓
RecursiveAction
```

Examples:

```java
class SumTask extends RecursiveTask<Long>
```

and:

```java
class PrintTask extends RecursiveAction
```

---

# 13. `ForkJoinPool` and Common Pool ⭐⭐⭐⭐⭐

Shared pool:

```java
ForkJoinPool.commonPool()
```

Dedicated pool:

```java
ForkJoinPool pool = new ForkJoinPool(4);
```

Use a custom pool when explicit isolation/control is required and the workload justifies it. Avoid casually placing blocking operations into a shared common pool.

---

# 14. Common Interview Traps 🚨

### Trap 1 — `fork()` creates a new thread

❌ Wrong:

```text
Every fork() = new Thread
```

✅ Correct:

```text
fork() = schedule ForkJoinTask in the pool
```

### Trap 2 — Work stealing means stealing running threads

❌ Wrong.

Workers steal **available tasks**, not another worker's currently executing Java stack.

### Trap 3 — Fork/Join always improves performance

❌ Wrong.

Small tasks can become slower because of task-management overhead.

### Trap 4 — Work stealing means no synchronization is needed

❌ Wrong.

Your task logic still needs to be thread-safe when tasks access shared mutable state.

### Trap 5 — `join()` automatically cancels everything else

❌ Wrong.

`join()` waits for a specific task; it does not automatically cancel unrelated tasks.

---

# 15. Shared Mutable State Warning ⭐⭐⭐⭐⭐

Avoid this pattern:

```java
static int total;
```

with multiple tasks directly doing:

```java
total += value;
```

That is not automatically thread-safe.

Prefer returning independent results:

```java
return leftResult + rightResult;
```

and combining them in the parent task.

This is one of the biggest strengths of divide-and-conquer design:

```text
independent computation
        ↓
local result
        ↓
combine
```

---

# 16. Fork/Join vs ExecutorService ⭐⭐⭐⭐⭐

| Fork/Join | ExecutorService |
|---|---|
| Recursive divide-and-conquer | General task execution |
| `RecursiveTask` / `RecursiveAction` | `Callable` / `Runnable` |
| Work stealing | Depends on executor; not its defining feature |
| Natural recursive splitting | Manual task decomposition |
| `ForkJoinPool` | ThreadPoolExecutor / other executors |

Interview answer:

> **I choose Fork/Join when the problem naturally decomposes into recursive independent subtasks. For general independent application tasks, ExecutorService is usually more straightforward.**

---

# 17. Fork/Join vs Parallel Streams ⭐⭐⭐⭐

Parallel streams provide a higher-level abstraction:

```java
list.parallelStream()
```

Fork/Join gives explicit control over:

```text
recursive decomposition
threshold
task boundaries
result combination
pool selection
```

Use Fork/Join when you need a custom recursive algorithm; use streams when the operation naturally fits the stream abstraction.

---

# 18. Blocking Work Warning 🚨

Be careful with:

```java
Thread.sleep(...);
databaseCall();
httpCall();
```

Long blocking operations can occupy Fork/Join workers and reduce available parallelism.

For I/O-heavy workloads, choose an execution model designed around that workload rather than assuming Fork/Join is automatically appropriate.

---

# 19. Complete Interview Coding Question 🏆

### Problem

> Given a large integer array, calculate the sum using Fork/Join. Split recursively, use a threshold, avoid shared mutable state, and combine results safely.

### Solution

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ForkJoinInterviewQuestion {

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

            SumTask left =
                    new SumTask(numbers, start, mid);

            SumTask right =
                    new SumTask(numbers, mid, end);

            left.fork();

            long rightResult = right.compute();
            long leftResult = left.join();

            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {

        int[] numbers = new int[1_000_000];

        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = i + 1;
        }

        ForkJoinPool pool = new ForkJoinPool(4);

        try {
            long result = pool.invoke(
                    new SumTask(numbers, 0, numbers.length)
            );

            System.out.println("Result = " + result);
        } finally {
            pool.shutdown();
        }
    }
}
```

---

# 20. Practice Challenge — Maximum ⭐⭐⭐⭐⭐

Try writing this **without looking at the solution**:

> Find the maximum value in a large array using `RecursiveTask<Integer>`.

Requirements:

```text
1. Use ForkJoinPool
2. Use a threshold
3. Split into two tasks
4. fork one branch
5. compute another branch
6. join forked branch
7. combine using Math.max()
8. Do not use shared mutable max
```

Expected design:

```java
return Math.max(leftMax, rightMax);
```

---

# 21. Practice Challenge — Tree Sum ⭐⭐⭐⭐⭐

Given:

```text
        10
       /  \
      5    7
     / \    \
    2   3    8
```

Write a `RecursiveTask<Integer>` that returns:

```text
35
```

Do not use a global/shared counter.

---

# 22. Practice Challenge — Recursive Search ⭐⭐⭐⭐

Implement:

```java
RecursiveTask<Integer>
```

that searches a large array and returns the index of the target or `-1`.

Think about:

```text
parallel branches
short-circuiting
cancellation
result precedence
```

Important interview point:

> Parallel search is more complicated than sequential search if you want to stop all unnecessary work immediately after finding a result.

---

# 23. 🏆 Complete Interview Code — Write From Memory

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class RecursiveTaskInterviewPractice {

    static class Task extends RecursiveTask<Long> {

        private static final int THRESHOLD = 1_000;

        private final int[] data;
        private final int start;
        private final int end;

        Task(int[] data, int start, int end) {
            this.data = data;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {

            // TODO 1: Base case
            if (end - start <= THRESHOLD) {
                long result = 0;
                for (int i = start; i < end; i++) {
                    result += data[i];
                }
                return result;
            }

            // TODO 2: Split
            int mid = (start + end) >>> 1;

            Task left = new Task(data, start, mid);
            Task right = new Task(data, mid, end);

            // TODO 3: Fork
            left.fork();

            // TODO 4: Compute
            long rightResult = right.compute();

            // TODO 5: Join
            long leftResult = left.join();

            // TODO 6: Combine
            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {

        int[] data = new int[100_000];

        for (int i = 0; i < data.length; i++) {
            data[i] = i + 1;
        }

        ForkJoinPool pool = new ForkJoinPool(4);

        try {
            long result = pool.invoke(
                    new Task(data, 0, data.length)
            );

            System.out.println(result);
        } finally {
            pool.shutdown();
        }
    }
}
```

If you can write this from memory, you understand the core recursive Fork/Join pattern.

---

# 24. 2-Minute Interview Answer 🏆

> **"Recursive tasks are the main programming model for Fork/Join divide-and-conquer algorithms. With `RecursiveTask<V>`, I implement `compute()`, check a threshold, and directly calculate when the problem is small. Otherwise I split the problem into subtasks, typically fork one branch, compute the other branch in the current worker, join the forked task, and combine the results. ForkJoinPool uses work stealing: when a worker has no local work, it can acquire available tasks from another worker's deque. This helps balance uneven recursive workloads. I avoid shared mutable state by returning partial results and combining them. I also choose the threshold carefully because too many tiny tasks can make Fork/Join slower than sequential execution."**

---

# 25. 30-Second Hinglish Answer

> **"Recursive Task ka use Fork/Join mein divide-and-conquer problem ke liye hota hai. `compute()` mein pehle threshold check karte hain. Task small hai to direct calculation, warna task ko split karte hain, ek branch `fork()`, doosri `compute()`, phir `join()` karke results combine karte hain. `ForkJoinPool` work stealing use karta hai, matlab idle worker kisi doosre worker ke available task ko steal karke execute kar sakta hai. Shared mutable state avoid karna aur correct threshold choose karna important hai."**

---

# 26. Rapid-Fire Interview Questions ⭐⭐⭐⭐⭐

### Q1. What is a recursive task?

A task that can split itself into smaller subtasks recursively and combine their results.

### Q2. What is work stealing?

Idle Fork/Join workers can take available tasks from other workers' queues.

### Q3. Does `fork()` create a thread?

No. It schedules a ForkJoinTask in the pool.

### Q4. Why use a threshold?

To control task granularity and avoid excessive task overhead.

### Q5. Why fork one and compute one?

It can reduce scheduling overhead and keeps the current worker productive.

### Q6. What is `RecursiveTask<V>`?

A recursive Fork/Join task that returns a value of type `V`.

### Q7. What is `RecursiveAction`?

A recursive task with no result.

### Q8. What is work stealing trying to solve?

Uneven distribution of recursive tasks and idle workers.

### Q9. Is Fork/Join suitable for blocking I/O?

Not generally as a default choice; blocking can occupy workers and hurt parallelism.

### Q10. Does work stealing make shared state thread-safe?

No.

### Q11. Can a task be split forever?

It should not be; use a meaningful base case/threshold.

### Q12. Why avoid global result variables?

They introduce synchronization/contention risks; independent results are easier to combine safely.

---

# 27. Quick Revision 🧠

```text
RecursiveTask
      ↓
compute()
      ↓
threshold?
 ┌────┴────┐
YES       NO
 ↓         ↓
direct    split
           ↓
        fork one
           ↓
       compute one
           ↓
          join
           ↓
        combine
```

```text
ForkJoinPool
      ↓
worker queues
      ↓
idle worker
      ↓
steal available task
      ↓
execute
```

### Golden Rules

```text
RecursiveTask<V> → result
RecursiveAction  → no result
fork()           → schedule
compute()        → execute
join()           → wait/get
threshold        → task granularity
work stealing    → load balancing
shared state     → still needs synchronization
```

---

# 28. 💻 Practice Checklist

- [ ] Write `RecursiveTask<Long>` from memory
- [ ] Write `RecursiveAction` from memory
- [ ] Implement parallel sum
- [ ] Implement parallel maximum
- [ ] Implement tree size
- [ ] Implement tree sum
- [ ] Implement parallel search
- [ ] Explain threshold
- [ ] Explain `fork()`
- [ ] Explain `join()`
- [ ] Explain work stealing
- [ ] Explain worker deque concept
- [ ] Explain why one branch is often forked and the other computed
- [ ] Explain shared-state risks
- [ ] Explain Fork/Join vs ExecutorService
- [ ] Explain Fork/Join vs parallel streams
- [ ] Explain blocking-work risk
- [ ] Write complete interview code without looking
- [ ] Give 2-minute interview answer
- [ ] Give 30-second Hinglish answer

---

## Navigation

[← 8.35 — Fork/Join Framework](../35-Fork-Join-Framework/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.37 — Parallel Streams & Concurrency Risks**