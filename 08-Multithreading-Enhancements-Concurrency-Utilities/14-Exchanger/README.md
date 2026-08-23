# 8.14 — `Exchanger`

> **Goal:** Understand how two threads can safely exchange objects at a synchronization point using `java.util.concurrent.Exchanger`.

---

## 1. What is `Exchanger`? ⭐⭐⭐⭐⭐

`Exchanger<V>` is a synchronization utility that allows **exactly two threads** to meet and exchange values of type `V`.

```text
Thread A                         Thread B
   |                                |
   | exchange("A-data")             | exchange("B-data")
   |-------------- rendezvous ------|
   |                                |
   ↓                                ↓
 receives "B-data"              receives "A-data"
```

### Memory Trick

> **Exchanger = two threads meet and swap values.**

---

# 2. Why `Exchanger`? ⭐⭐⭐⭐⭐

Use it when two threads need a direct hand-off or swap of data at a synchronization point.

Typical examples:

- Producer ↔ Consumer hand-off
- Double-buffering
- Pipeline stages
- Batch/data exchange between two worker threads
- Temporary ownership transfer

---

# 3. Basic Example ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Exchanger;

public class BasicExchangerExample {

    public static void main(String[] args) {

        Exchanger<String> exchanger = new Exchanger<>();

        Thread producer = new Thread(() -> {
            try {
                String received = exchanger.exchange("Data from Producer");
                System.out.println("Producer received: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                String received = exchanger.exchange("Data from Consumer");
                System.out.println("Consumer received: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

Expected conceptually:

```text
Producer sends → Data from Producer
Consumer sends → Data from Consumer

Producer receives → Data from Consumer
Consumer receives → Data from Producer
```

The exact output order can vary because thread scheduling is nondeterministic.

---

# 4. `exchange()` ⭐⭐⭐⭐⭐

The main operation is:

```java
V exchange(V value) throws InterruptedException
```

The calling thread waits until another thread arrives at the same exchanger.

Then both threads exchange their values and continue.

---

# 5. `exchange()` Is a Rendezvous ⭐⭐⭐⭐⭐

Think of it as a meeting point:

```text
Thread A
   |
   | exchange(A)
   ↓
 [WAIT]
   ↑
   | exchange(B)
Thread B
```

Once both arrive:

```text
A receives B
B receives A
```

### Important

`Exchanger` is not a general-purpose queue.

It synchronizes **pairs of threads**.

---

# 6. Timed `exchange()` ⭐⭐⭐⭐⭐

You can avoid waiting indefinitely:

```java
String result = exchanger.exchange(
        "data",
        2,
        TimeUnit.SECONDS
);
```

If another participant does not arrive within the timeout, the call throws `TimeoutException`.

Example:

```java
try {
    String result = exchanger.exchange(
            "data",
            2,
            TimeUnit.SECONDS
    );
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} catch (TimeoutException e) {
    System.out.println("Exchange timed out");
}
```

---

# 7. Interruption ⭐⭐⭐⭐⭐

Both forms of `exchange()` are interruptible.

```java
try {
    String result = exchanger.exchange(data);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### Best Practice

When catching `InterruptedException` and you cannot propagate it:

```java
Thread.currentThread().interrupt();
```

This restores the thread's interrupt status.

---

# 8. Generic Type ⭐⭐⭐⭐

`Exchanger` is generic:

```java
Exchanger<String> stringExchanger = new Exchanger<>();
```

For integers:

```java
Exchanger<Integer> integerExchanger = new Exchanger<>();
```

For custom objects:

```java
Exchanger<Order> orderExchanger = new Exchanger<>();
```

---

# 9. Practice — Producer / Consumer Handoff ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Exchanger;

public class ProducerConsumerExchanger {

    public static void main(String[] args) {

        Exchanger<String> exchanger = new Exchanger<>();

        Thread producer = new Thread(() -> {
            String buffer = "Batch-1";

            try {
                System.out.println("Producer prepared: " + buffer);
                buffer = exchanger.exchange(buffer);
                System.out.println("Producer received buffer: " + buffer);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            String buffer = "Empty-Buffer";

            try {
                Thread.sleep(500);
                buffer = exchanger.exchange(buffer);
                System.out.println("Consumer received: " + buffer);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

### Mental Model

```text
Producer owns → Batch-1
Consumer owns → Empty-Buffer

        exchange()
           ↓
Producer → Empty-Buffer
Consumer → Batch-1
```

---

# 10. Practice — Double Buffering ⭐⭐⭐⭐⭐

A useful conceptual application is swapping two buffers between a producer and consumer.

```java
import java.util.concurrent.Exchanger;

public class DoubleBufferExample {

    public static void main(String[] args) {

        Exchanger<String> exchanger = new Exchanger<>();

        Thread producer = new Thread(() -> {
            String buffer = "Producer-Buffer";

            try {
                for (int i = 1; i <= 3; i++) {
                    buffer = "Batch-" + i;
                    System.out.println("Producer fills: " + buffer);

                    buffer = exchanger.exchange(buffer);

                    System.out.println("Producer gets back: " + buffer);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            String buffer = "Consumer-Buffer";

            try {
                for (int i = 1; i <= 3; i++) {
                    buffer = exchanger.exchange(buffer);

                    System.out.println("Consumer processes: " + buffer);

                    buffer = "Reused-Consumer-Buffer";
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

### Key Idea

The same two participants can repeatedly exchange buffers.

---

# 11. `Exchanger` Is Reusable ⭐⭐⭐⭐

Unlike `CountDownLatch`, an `Exchanger` can be used repeatedly.

```text
Exchange 1
A ↔ B

Exchange 2
A ↔ B

Exchange 3
A ↔ B
```

Each exchange is a new rendezvous.

---

# 12. What Happens If Only One Thread Arrives? ⭐⭐⭐⭐⭐

```java
Exchanger<String> exchanger = new Exchanger<>();

exchanger.exchange("hello");
```

The thread waits because no partner has arrived.

```text
Thread A
   ↓
exchange(A)
   ↓
WAITING
   ↓
needs Thread B
```

Use timed `exchange()` if indefinite waiting is unacceptable.

---

# 13. Practice — Timeout ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ExchangerTimeoutExample {

    public static void main(String[] args) {

        Exchanger<String> exchanger = new Exchanger<>();

        Thread thread = new Thread(() -> {
            try {
                String result = exchanger.exchange(
                        "Hello",
                        2,
                        TimeUnit.SECONDS
                );

                System.out.println("Received: " + result);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } catch (TimeoutException e) {
                System.out.println("No partner arrived in time");
            }
        });

        thread.start();
    }
}
```

---

# 14. Practice — Custom Object Exchange ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.Exchanger;

public class CustomObjectExchange {

    static class Message {
        private final String sender;
        private final String text;

        Message(String sender, String text) {
            this.sender = sender;
            this.text = text;
        }

        @Override
        public String toString() {
            return sender + ": " + text;
        }
    }

    public static void main(String[] args) {

        Exchanger<Message> exchanger = new Exchanger<>();

        Thread serviceA = new Thread(() -> {
            try {
                Message response = exchanger.exchange(
                        new Message("Service-A", "Request-A")
                );

                System.out.println("Service-A received: " + response);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread serviceB = new Thread(() -> {
            try {
                Message response = exchanger.exchange(
                        new Message("Service-B", "Response-B")
                );

                System.out.println("Service-B received: " + response);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        serviceA.start();
        serviceB.start();
    }
}
```

---

# 15. `Exchanger` Does Not Need a Queue ⭐⭐⭐⭐⭐

A blocking queue stores items until another thread consumes them.

```text
BlockingQueue
Producer → Queue → Consumer
```

An exchanger directly synchronizes two participants:

```text
Thread A ↔ Exchanger ↔ Thread B
```

### Interview Point

> `Exchanger` is a direct two-party handoff mechanism, whereas a `BlockingQueue` is a storage-and-consumption mechanism.

---

# 16. `Exchanger` vs `BlockingQueue` ⭐⭐⭐⭐⭐

| Feature | `Exchanger` | `BlockingQueue` |
|---|---|---|
| Main purpose | Pairwise exchange | Producer-consumer queue |
| Participants | Two at a time | Many possible |
| Stores items | No persistent queue | Yes |
| Exchange operation | `exchange()` | `put()` / `take()` |
| Direct handoff | Yes | Usually no |
| Reusable | Yes | Yes |
| Best for | Pairwise swap | Work distribution |

### Memory Trick

> **Exchanger = swap. Queue = store.**

---

# 17. `Exchanger` vs `CountDownLatch` ⭐⭐⭐⭐⭐

| Feature | `Exchanger` | `CountDownLatch` |
|---|---|---|
| Purpose | Exchange values | Wait for count to reach zero |
| Value transfer | Yes | No |
| Reusable | Yes | No |
| Participants | Two-party exchange | One or many waiting threads |
| Main method | `exchange()` | `await()` / `countDown()` |

---

# 18. `Exchanger` vs `CyclicBarrier` ⭐⭐⭐⭐⭐

| Feature | `Exchanger` | `CyclicBarrier` |
|---|---|---|
| Main purpose | Swap values | Synchronize parties |
| Data exchange | Yes | No |
| Number of parties | Exactly two per exchange | Configured number of parties |
| Reusable | Yes | Yes |
| Main operation | `exchange()` | `await()` |

### Memory Trick

```text
Exchanger
→ "Give me your value and take mine."

Barrier
→ "Wait until everyone reaches this point."
```

---

# 19. `Exchanger` vs `Semaphore` ⭐⭐⭐⭐⭐

| Feature | `Exchanger` | `Semaphore` |
|---|---|---|
| Main purpose | Pairwise data exchange | Limit concurrency |
| Controls permits | No | Yes |
| Transfers values | Yes | No |
| Typical use | Buffer handoff | Resource capacity |

---

# 20. Important Concept — Exchange Is Atomic at the Rendezvous ⭐⭐⭐⭐

From the application's perspective, once both participants successfully complete `exchange()`, each receives the other participant's value.

```text
Before:
A owns X
B owns Y

After:
A owns Y
B owns X
```

The method establishes the handoff as part of the synchronization operation.

---

# 21. Practice — Two-Stage Pipeline ⭐⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class PipelineExchangeExample {

    public static void main(String[] args) {

        Exchanger<Integer> exchanger = new Exchanger<>();

        Thread stageOne = new Thread(() -> {
            try {
                int value = 10;

                for (int i = 0; i < 3; i++) {
                    value = value * 2;

                    System.out.println("Stage 1 produced: " + value);

                    value = exchanger.exchange(value);
                }

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread stageTwo = new Thread(() -> {
            try {
                int value = 0;

                for (int i = 0; i < 3; i++) {
                    value = exchanger.exchange(value);

                    System.out.println("Stage 2 received: " + value);
                    value = value + 1;
                }

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        stageOne.start();
        stageTwo.start();
    }
}
```

---

# 22. Important Pitfall — Partner Mismatch ⚠️ ⭐⭐⭐⭐

An exchanger pairs whichever two threads arrive for an exchange operation.

Do not assume a specific thread will be paired unless your design guarantees the participant structure.

For example, multiple independent producers and consumers should not casually share one exchanger if each pair must communicate with a specific partner.

### Rule

> Use one exchanger as a rendezvous point for a clearly defined pairwise protocol.

---

# 23. Important Pitfall — Repeated Exchange Must Be Balanced ⚠️

If one thread performs:

```java
for (int i = 0; i < 3; i++) {
    exchanger.exchange(data);
}
```

but the partner performs only one exchange, later iterations can block indefinitely.

```text
Thread A: exchange → exchange → exchange
Thread B: exchange
                   ↑
             A waits here
```

### Rule

Both sides need compatible exchange protocols.

---

# 24. Important Pitfall — Timeout Handling ⚠️

When using timed exchange:

```java
try {
    exchanger.exchange(data, 1, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    // partner did not arrive in time
}
```

Do not silently ignore the timeout. Decide whether to:

- retry,
- abort the operation,
- fall back to another path,
- log/measure the failure.

---

# 25. Happens-Before Insight ⭐⭐⭐⭐⭐

A successful exchange provides synchronization between the two participating threads.

This means actions before a successful `exchange()` in one thread are visible to the other participant after its corresponding successful `exchange()`.

### Interview Version

> A successful `Exchanger.exchange()` establishes the necessary synchronization relationship for the exchanged handoff, so it can be used as a communication boundary between the two participants.

---

# 26. Practice — ExecutorService Integration ⭐⭐⭐⭐

```java
import java.util.concurrent.*;

public class ExchangerExecutorExample {

    public static void main(String[] args) throws Exception {

        Exchanger<String> exchanger = new Exchanger<>();

        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<?> first = executor.submit(() -> {
            try {
                String value = exchanger.exchange("Worker-A data");
                System.out.println("A received: " + value);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Future<?> second = executor.submit(() -> {
            try {
                String value = exchanger.exchange("Worker-B data");
                System.out.println("B received: " + value);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        first.get();
        second.get();

        executor.shutdown();
    }
}
```

### Important

A thread pool must have enough available workers for both participants to reach the exchanger. A poorly sized or saturated executor can make the exchange appear to hang.

---

# 27. Thread-Pool Starvation Pitfall ⭐⭐⭐⭐⭐

Consider a single-thread executor:

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

If task A calls:

```java
exchanger.exchange(data);
```

and task B is waiting in the executor queue, task A occupies the only worker and waits for B.

B cannot start.

```text
1 worker
   ↓
Task A runs
   ↓
exchange()
   ↓
WAIT
   ↓
Task B cannot start ❌
```

### Lesson

> Blocking synchronization inside a thread pool requires enough executor capacity for all required participants.

---

# 28. Practice — Demonstrate the Pool-Sizing Issue ⭐⭐⭐⭐⭐

Avoid this design:

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

executor.submit(() -> {
    try {
        exchanger.exchange("A");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

executor.submit(() -> {
    try {
        exchanger.exchange("B");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});
```

Use at least two workers for the two participants:

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
```

This is a valuable production/interview debugging scenario.

---

# 29. Common Mistakes ❌

### Mistake 1 — Treating `Exchanger` as a queue

It is a rendezvous, not persistent storage.

### Mistake 2 — Having only one participant

The thread can wait indefinitely.

### Mistake 3 — Mismatched repeated exchanges

One side performs more exchanges than the other.

### Mistake 4 — Ignoring interruption

`exchange()` is interruptible.

### Mistake 5 — Ignoring timeout

Timed exchange can fail with `TimeoutException`.

### Mistake 6 — Using it for many-to-many work distribution

Use `BlockingQueue` or another appropriate concurrency abstraction.

### Mistake 7 — Insufficient thread-pool capacity

A participant can block while another participant cannot obtain an executor worker.

### Mistake 8 — Assuming a specific partner without protocol guarantees

The design must clearly define which two threads should exchange.

---

# 30. Interview Scenarios ⭐⭐⭐⭐⭐

### Q1. What is `Exchanger`?

> `Exchanger` is a Java concurrency utility that lets two threads meet at a synchronization point and exchange values.

### Q2. How many threads participate in one exchange?

> Exactly two threads participate in a successful exchange operation.

### Q3. What happens if the partner does not arrive?

> The non-timed `exchange()` waits; timed exchange throws `TimeoutException` if the timeout expires.

### Q4. Is `Exchanger` reusable?

> Yes. The same exchanger can coordinate repeated exchanges.

### Q5. `Exchanger` vs `BlockingQueue`?

> Exchanger directly swaps values between two participants; a blocking queue stores items and supports producer-consumer workflows with potentially many participants.

### Q6. `Exchanger` vs `CountDownLatch`?

> Exchanger transfers values between two participants and is reusable; CountDownLatch coordinates completion/signaling and is one-shot.

### Q7. `Exchanger` vs `CyclicBarrier`?

> Exchanger swaps values between two threads; CyclicBarrier makes a configured group wait at a common barrier.

### Q8. Is `exchange()` interruptible?

> Yes. It can throw `InterruptedException`.

### Q9. Why use timed `exchange()`?

> To prevent indefinite waiting when the partner may fail to arrive.

### Q10. Can `Exchanger` be used with custom objects?

> Yes. It is generic and can exchange any reference type.

### Q11. Does `Exchanger` store values for later?

> No. It is a rendezvous mechanism; both participants meet and exchange during the operation.

### Q12. Give a real-world use case.

> A producer and consumer can exchange full and empty buffers, enabling a double-buffering pipeline.

### Q13. What happens with repeated exchanges?

> Each call creates another rendezvous. Both participants need compatible exchange counts/protocols.

### Q14. Can a thread pool cause an Exchanger deadlock/starvation problem?

> Yes. If a worker blocks at `exchange()` while the pool lacks capacity to run the partner, the partner may never execute.

### Q15. What is the key difference from a semaphore?

> Exchanger transfers values between two participants; semaphore controls the number of concurrent permit holders.

---

# 31. Quick Revision

```text
Exchanger<V>
      ↓
Two participants
      ↓
exchange(value)
      ↓
Rendezvous
      ↓
A receives B's value
B receives A's value
```

### Key APIs

```java
new Exchanger<T>()
exchange(value)
exchange(value, timeout, unit)
```

### Key facts ⭐

```text
✓ Exactly two participants per successful exchange
✓ Direct value handoff
✓ Reusable
✓ Generic
✓ Blocking by default
✓ Interruptible
✓ Timed exchange supported
✓ Useful for double buffering
✓ Not a queue
✓ Not a concurrency limiter
```

---

# 🏆 2-Minute Interview Answer

> **"`Exchanger` is a Java concurrency utility from `java.util.concurrent` that allows two threads to meet at a synchronization point and exchange values. Each thread calls `exchange(value)`, and the call waits until another participant arrives. Once both arrive, each receives the other thread's value. It is useful for direct handoff patterns such as producer-consumer buffer exchange and double buffering. `Exchanger` is reusable, supports generic values, and also provides a timed `exchange()` to avoid waiting indefinitely. It is different from a `BlockingQueue` because it does not provide persistent storage; it directly synchronizes a pair of participants. It is also different from `CyclicBarrier`, which synchronizes a group without exchanging values. In production, we should handle interruption, timeouts and executor capacity carefully because a participant can block while waiting for its partner."**

---

# 💻 Practice Checklist

- [ ] Create an `Exchanger<String>`.
- [ ] Exchange values between two threads.
- [ ] Practice repeated exchanges.
- [ ] Practice timed `exchange()`.
- [ ] Handle `InterruptedException`.
- [ ] Handle `TimeoutException`.
- [ ] Exchange custom objects.
- [ ] Build producer-consumer buffer handoff.
- [ ] Build a double-buffering example.
- [ ] Integrate with `ExecutorService`.
- [ ] Understand executor starvation risk.
- [ ] Compare `Exchanger` vs `BlockingQueue`.
- [ ] Compare `Exchanger` vs `CountDownLatch`.
- [ ] Compare `Exchanger` vs `CyclicBarrier`.
- [ ] Compare `Exchanger` vs `Semaphore`.
- [ ] Explain rendezvous semantics.
- [ ] Explain why one participant can block.
- [ ] Explain timed exchange.
- [ ] Explain the topic in under 2 minutes.

---

## Navigation

[← 8.13 — `Semaphore`](../13-Semaphore/README.md)

[🏠 Chapter 8](../README.md)

[🏠 Core Java Master README](../../README.md)

**Next → 8.15 — `Phaser`**