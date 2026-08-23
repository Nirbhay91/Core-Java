# Chapter 8 — Multithreading Enhancements / Concurrency Utilities

> **Goal:** Move from low-level thread management to production-ready Java concurrency utilities.
>
> Focus: thread pools, executors, scheduling, futures, synchronization utilities, concurrent collections, locks, atomics, Fork/Join and `CompletableFuture`.

## 📚 Chapter 8 Roadmap

| # | Topic | Status | Link |
|---:|---|---|---|
| 8.1 | `Executor` and `ExecutorService` Fundamentals | ✅ Completed | [Open](01-Executor-and-ExecutorService-Fundamentals/README.md) |
| 8.2 | Thread Pool Fundamentals | ✅ Completed | [Open](02-Thread-Pool-Fundamentals/README.md) |
| 8.3 | `execute()` vs `submit()` | ✅ Completed | [Open](03-execute-vs-submit/README.md) |
| 8.4 | `shutdown()` vs `shutdownNow()` | ✅ Completed | [Open](04-shutdown-vs-shutdownNow/README.md) |
| 8.5 | `Future` and `Callable` | ✅ Completed | [Open](05-Future-and-Callable/README.md) |
| 8.6 | `ScheduledExecutorService` | ⏳ Pending | — |
| 8.7 | Thread Pool Types | ⏳ Pending | — |
| 8.8 | `ThreadPoolExecutor` Internals | ⏳ Pending | — |
| 8.9 | Queueing, Rejection Policies & Backpressure | ⏳ Pending | — |
| 8.10 | Custom `ThreadFactory` | ⏳ Pending | — |
| 8.11 | `CountDownLatch` | ⏳ Pending | — |
| 8.12 | `CyclicBarrier` | ⏳ Pending | — |
| 8.13 | `Semaphore` | ⏳ Pending | — |
| 8.14 | `Exchanger` | ⏳ Pending | — |
| 8.15 | `Phaser` | ⏳ Pending | — |
| 8.16 | `ReentrantLock` | ⏳ Pending | — |
| 8.17 | `tryLock()` and Timed Locking | ⏳ Pending | — |
| 8.18 | `ReentrantReadWriteLock` | ⏳ Pending | — |
| 8.19 | `StampedLock` | ⏳ Pending | — |
| 8.20 | `Condition` | ⏳ Pending | — |
| 8.21 | Atomic Variables & CAS | ⏳ Pending | — |
| 8.22 | `LongAdder` / `LongAccumulator` | ⏳ Pending | — |
| 8.23 | Concurrent Collections Overview | ⏳ Pending | — |
| 8.24 | `ConcurrentHashMap` Deep Dive | ⏳ Pending | — |
| 8.25 | `CopyOnWriteArrayList` | ⏳ Pending | — |
| 8.26 | `BlockingQueue` Implementations | ⏳ Pending | — |
| 8.27 | `ArrayBlockingQueue` vs `LinkedBlockingQueue` | ⏳ Pending | — |
| 8.28 | `PriorityBlockingQueue` / `DelayQueue` | ⏳ Pending | — |
| 8.29 | `ConcurrentLinkedQueue` | ⏳ Pending | — |
| 8.30 | `CompletableFuture` Fundamentals | ⏳ Pending | — |
| 8.31 | `thenApply` / `thenCompose` / `thenCombine` | ⏳ Pending | — |
| 8.32 | Exception Handling in `CompletableFuture` | ⏳ Pending | — |
| 8.33 | Async Execution & Custom Executors | ⏳ Pending | — |
| 8.34 | `allOf()` / `anyOf()` | ⏳ Pending | — |
| 8.35 | Fork/Join Framework | ⏳ Pending | — |
| 8.36 | Recursive Tasks & Work Stealing | ⏳ Pending | — |
| 8.37 | Parallel Streams & Concurrency Risks | ⏳ Pending | — |
| 8.38 | Thread Pool Sizing & Performance | ⏳ Pending | — |
| 8.39 | Graceful Shutdown & Production Patterns | ⏳ Pending | — |
| 8.40 | Concurrency Utilities Interview Scenarios | ⏳ Pending | — |
| 8.41 | Concurrency Utilities Quick Revision | ⏳ Pending | — |
| 8.42 | Concurrency Utilities Final Assessment | ⏳ Pending | — |

---

# 🧠 Practice Code Strategy

Every topic will include runnable Java practice wherever applicable:

```text
Concept → Runnable Example → Wrong/Unsafe Version → Correct Version
       → Production Scenario → Interview Questions → Debugging → Revision
```

---

## Navigation

[🏠 Core Java Master README](../README.md)

**Current → 8.5 — `Future` and `Callable` → ✅ Completed**

**Next → 8.6 — `ScheduledExecutorService`**