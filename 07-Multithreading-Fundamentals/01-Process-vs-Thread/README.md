# 7.1 — Process vs Thread

## 🎯 Objective

Understand the difference between a **Process** and a **Thread**, why Java applications use threads, and how this distinction forms the foundation of Java multithreading.

---

## 1. What is a Process?

A **process** is an independent program in execution.

When an application starts, the operating system creates a process and gives it resources such as:

- Virtual address space
- Heap / memory resources
- File handles
- OS resources
- Security context
- At least one execution thread

### Example

If you start a Java application:

```bash
java MyApplication
```

The operating system creates a process for that Java application. The JVM runs inside that process.

```text
Operating System
       │
       └── Java Process
              │
              └── JVM
                    ├── Heap
                    ├── Method Area / Metaspace
                    ├── JVM runtime structures
                    └── Threads
```

---

## 2. What is a Thread?

A **thread is an execution path within a process**.

A process can contain multiple threads that execute concurrently and share many resources belonging to that process.

```text
Process
│
├── Thread-1
├── Thread-2
├── Thread-3
└── Thread-4
```

In Java, the JVM starts with multiple JVM-managed/system threads, and application code can create additional threads.

---

## 3. Process vs Thread

| Feature | Process | Thread |
|---|---|---|
| Definition | Independent executing program | Execution path within a process |
| Memory | Has its own virtual address space | Shares process memory/resources |
| Isolation | Higher | Lower |
| Creation cost | Generally higher | Generally lower |
| Context switch | Generally more expensive | Generally cheaper |
| Communication | IPC mechanisms commonly required | Shared memory can be used |
| Failure isolation | Better | Weaker; thread failure can affect process |
| Ownership | OS-level resource container | Execution unit inside process |
| Java example | JVM process | `Thread` / application thread |

> **Interview nuance:** "Threads are lightweight processes" is a common simplification. A better answer is that threads are execution units within a process and generally share the process's address space and resources.

---

## 4. Process Memory Isolation

Processes normally have separate virtual address spaces.

```text
Process A
┌──────────────────────┐
│ Heap                 │
│ Code                 │
│ Data                 │
└──────────────────────┘

Process B
┌──────────────────────┐
│ Heap                 │
│ Code                 │
│ Data                 │
└──────────────────────┘
```

A normal memory access from Process A cannot directly access Process B's memory.

Communication therefore generally uses mechanisms such as:

- Pipes
- Sockets
- Shared memory
- Message queues
- Other OS IPC mechanisms

---

## 5. Thread Memory Sharing

Threads within the same process share process-level memory/resources, but each thread has its own execution state.

Conceptually:

```text
Java Process
│
├── Shared
│   ├── Heap
│   └── Class metadata / JVM-managed shared areas
│
├── Thread-1
│   ├── Stack
│   ├── Program counter
│   └── Execution state
│
└── Thread-2
    ├── Stack
    ├── Program counter
    └── Execution state
```

This shared-memory model makes communication between threads convenient, but it introduces concurrency problems such as:

- Race conditions
- Visibility problems
- Atomicity problems
- Ordering problems
- Deadlocks

These become important in the next multithreading topics.

---

## 6. Why Do We Need Multiple Threads?

### 1. Concurrency

Multiple independent tasks can make progress during overlapping periods.

Example:

```text
Main Thread
    │
    ├── Process request
    ├── Start background task
    └── Continue other work
```

### 2. Responsiveness

A long-running operation can be separated from the thread responsible for responsive interaction.

### 3. Parallelism

On a multicore system, multiple threads may execute simultaneously on different CPU cores.

```text
CPU Core 1 → Thread A
CPU Core 2 → Thread B
CPU Core 3 → Thread C
```

> **Concurrency and parallelism are not identical.** Concurrency is about managing multiple tasks with overlapping progress; parallelism means tasks are actually executing simultaneously.

---

## 7. Java Thread Model

Java exposes threads primarily through `java.lang.Thread` and related concurrency APIs.

Basic example:

```java
public class Demo {

    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            System.out.println("Worker thread: "
                    + Thread.currentThread().getName());
        });

        worker.start();
    }
}
```

Conceptually:

```text
main()
  │
  ├── creates Thread object
  │
  └── start()
        │
        └── JVM/OS scheduling
                │
                └── worker executes
```

The important point is that calling `start()` requests a new thread of execution; calling `run()` directly does not create that new execution thread.

---

## 8. Process Creation vs Thread Creation

```text
Process creation
    ↓
OS creates an isolated process environment
    ↓
Process executes program

Thread creation
    ↓
Thread is created inside an existing process
    ↓
Thread shares process resources
```

In Java application development, you normally create application threads within the JVM process rather than manually creating OS processes for each task.

For process-level execution Java also provides APIs such as `ProcessBuilder` when an application needs to launch another operating-system process.

---

## 9. Important Interview Concept — Shared vs Private State

A strong interview answer should avoid saying **"threads share everything."**

Instead:

### Shared / process-level resources

Threads generally share:

- Heap objects
- Class metadata and static state
- Open resources associated with the process
- Other process-level resources

### Per-thread execution state

Each thread has its own:

- Stack
- Program counter
- Current execution context

This distinction explains why two threads can access the same object but maintain separate local variables.

Example:

```java
class Counter {
    int value;
}

Counter counter = new Counter();

Thread t1 = new Thread(() -> counter.value++);
Thread t2 = new Thread(() -> counter.value++);
```

Both threads can access the same `counter` object because it is a shared heap object.

But the increment operation is not automatically safe just because each thread has its own stack.

---

## 10. Process vs Thread — Simple Analogy

Think of a **process as a company** and **threads as employees working inside that company**.

```text
Company = Process

Employees = Threads

Shared office/resources = Process resources

Employee's personal desk/work state = Thread-specific execution state
```

Employees can collaborate easily because they work in the same organization, but shared resources can cause conflicts if they are not coordinated.

---

## 11. Common Interview Trap

### ❌ Weak answer

> "A process is heavy and a thread is lightweight."

This is incomplete.

### ✅ Better answer

> "A process is an independent execution environment with its own virtual address space and OS-managed resources. A thread is an execution unit within a process. Threads in the same process generally share memory and process resources while maintaining their own execution state such as stack and program counter. This sharing makes thread communication efficient but also introduces synchronization and visibility concerns."

---

## 12. Concurrency vs Parallelism

### Concurrency

Multiple tasks make progress over overlapping periods.

```text
Time →

Task A: ███   ███
Task B:   ███   ███
```

### Parallelism

Multiple tasks execute at the same time on different execution resources.

```text
Core 1: █████████
Core 2: █████████
```

A single-core system can support concurrency through scheduling, while true hardware parallelism requires multiple execution resources.

---

## 13. Why Shared Memory Creates Problems

Suppose two threads execute:

```java
counter++;
```

It looks like one operation but conceptually involves reading, modifying and writing the value.

```text
Read
 ↓
Add 1
 ↓
Write
```

Two threads can interleave these steps and lose an update.

```text
Thread A → Read 10
Thread B → Read 10
Thread A → Write 11
Thread B → Write 11

Expected = 12
Actual   = 11
```

This is a **race condition**.

The next topics will cover critical sections, synchronization, atomicity and visibility.

---

## 14. Java-Specific Memory Note

Do not interpret the simplified diagram as a complete JVM memory specification.

The Java Memory Model (JMM) defines rules for visibility, ordering and synchronization between threads. JVM implementations also have runtime-specific details.

For interview purposes, remember:

```text
Process
  ↓
JVM
  ↓
Multiple Java Threads
  ↓
Shared heap objects + per-thread execution state
  ↓
JMM governs visibility / ordering guarantees
```

---

## 15. When Would You Use Processes Instead of Threads?

Processes are useful when stronger isolation is desirable or when running an independent application/service is appropriate.

Examples:

- Running another executable
- Strong fault isolation
- Different runtime environments
- Independent deployment boundaries
- Security isolation requirements

Threads are useful when tasks naturally belong to the same application process and need efficient access to shared in-process state.

---

## 16. Interview Questions

### Q1. What is a process?

A process is an independent program in execution with its own virtual address space and OS-managed resources.

### Q2. What is a thread?

A thread is an execution unit within a process. Threads generally share process resources while maintaining their own execution state.

### Q3. Can one process have multiple threads?

Yes. A process can contain multiple threads that execute concurrently and share process resources.

### Q4. Do threads share memory?

Threads in the same process generally share the process's heap and other process-level resources, while each thread has its own stack and execution state.

### Q5. Why are threads useful?

They enable concurrent execution, responsiveness and potentially parallel execution on multicore systems.

### Q6. What is the difference between concurrency and parallelism?

Concurrency means multiple tasks can make overlapping progress; parallelism means multiple tasks execute simultaneously on separate execution resources.

### Q7. Why can threads cause race conditions?

Because multiple threads can access and modify shared mutable state without sufficient synchronization.

### Q8. Are threads always faster than processes?

No. Performance depends on workload, scheduling, synchronization, communication and system architecture. Threads generally have lower overhead for in-process concurrency, but they are not automatically faster.

### Q9. Is a Java thread the same as an OS thread?

Modern JVMs commonly map platform Java threads to native OS threads, but the exact implementation is JVM/platform dependent. Java also has virtual threads in modern Java, which use a different scheduling model.

### Q10. What is the relationship between JVM, process and thread?

A Java application normally runs in a JVM hosted inside an OS process. The JVM manages Java execution, including Java threads.

---

## 17. 2-Minute Interview Answer ⭐⭐⭐⭐⭐

> **"A process is an independent execution environment with its own virtual address space and OS-managed resources. A thread is an execution unit inside a process. Multiple threads within the same process generally share heap objects and process resources, while each thread has its own stack and execution state. Threads are useful for concurrency, responsiveness and parallelism on multicore systems. The main advantage of shared process memory is efficient communication, but it also introduces race conditions, visibility and synchronization problems. In Java, threads can be created using Thread or Runnable, although higher-level Executor APIs are preferred for many production use cases. Also, concurrency and parallelism are different: concurrency is overlapping progress, while parallelism is simultaneous execution."

---

## 18. Quick Revision

```text
Process
  ↓
Independent execution environment
  ↓
Own virtual address space
  ↓
Contains threads

Thread
  ↓
Execution unit inside process
  ↓
Shares process resources
  ↓
Own stack + execution state

Shared memory
  ↓
Efficient communication
  ↓
But creates concurrency problems
  ↓
Race condition / Visibility / Atomicity / Ordering
```

### One-line memory trick

> **Process = isolated execution environment; Thread = execution path inside that environment.**

---

## 19. Completion Checklist

- [x] Process definition
- [x] Thread definition
- [x] Process vs Thread comparison
- [x] Process memory isolation
- [x] Thread memory sharing
- [x] Java thread model
- [x] Shared vs thread-specific state
- [x] Concurrency vs parallelism
- [x] Race-condition introduction
- [x] Java Memory Model context
- [x] Interview questions
- [x] 2-minute interview answer
- [x] Quick revision

---

## Navigation

[← Chapter 7 — Multithreading Fundamentals](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 7.2 — Thread Creation using `Thread` class**