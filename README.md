# Core Java — Interview Preparation Roadmap

A chapter-wise roadmap for **Core Java + Java Interview Preparation**. Each topic will be studied and completed one-by-one with a consistent structure: concept → internals → examples → code → interview questions → revision.

## 🎯 Learning Sequence

| # | Topic | Status | Link |
|---:|---|---|---|
| 1 | Java Language Fundamentals | ⏳ Pending | — |
| 2 | Operators & Assignments | ⏳ Pending | — |
| 3 | Flow Control | ⏳ Pending | — |
| 4 | Declarations & Access Modifiers | ⏳ Pending | — |
| 5 | OOPs | ⏳ Pending | — |
| 6 | Exception Handling | ⏳ Pending | — |
| 7 | **Multithreading Fundamentals** | 🚧 In Progress | [Open](07-Multithreading-Fundamentals/README.md) |
| 8 | Multithreading Enhancements / Concurrency Utilities | ⏳ Pending | — |
| 9 | Inner Classes | ⏳ Pending | — |
| 10 | `java.lang` Package | ⏳ Pending | — |
| 11 | File I/O | ⏳ Pending | — |
| 12 | Serialization | ⏳ Pending | — |
| 13 | Regular Expressions | ⏳ Pending | — |
| 14 | Collections Framework | ⏳ Pending | — |
| 15 | Generics | ⏳ Pending | — |
| 16 | Garbage Collection | ⏳ Pending | — |
| 17 | Enum | ⏳ Pending | — |
| 18 | Internationalization (i18n) | ⏳ Pending | — |
| 19 | Development / Java Platform Essentials | ⏳ Pending | — |
| 20 | Assertions | ⏳ Pending | — |
| 21 | JVM Architecture | ⏳ Pending | — |
| 22 | Java 8+ New Features | ⏳ Pending | — |
| 23 | String | ⏳ Pending | — |

---

# Chapter 1 — Java Language Fundamentals

### 1. Introduction
- What is Java?
- Java features
- JDK vs JRE vs JVM
- Java compilation and execution flow

### 2. Identifiers
- What is an identifier?
- Rules for defining identifiers
- Valid vs invalid identifiers
- Naming conventions

### 3. Reserved Words / Keywords
- Data type keywords
- Flow-control keywords
- Modifier keywords
- Exception-handling keywords
- Class/interface-related keywords
- Object-related keywords
- `void`
- Unused/reserved keywords
- Reserved literals
- `enum`

### 4. Data Types
- Primitive vs reference types
- Integral types: `byte`, `short`, `int`, `long`
- Floating-point types: `float`, `double`
- `boolean`
- `char`
- Primitive ranges and defaults
- Is Java purely object-oriented?

### 5. Literals
- Integral literals
- Floating-point literals
- Boolean literals
- Character literals
- String literals
- Binary literals
- Underscores in numeric literals

### 6. Arrays
- Array introduction
- Declaration
- Single-dimensional arrays
- Two-dimensional arrays
- Multi-dimensional arrays
- Construction
- Initialization
- Declaration + construction + initialization
- `length` vs `length()`
- Anonymous arrays
- Array element assignment
- Array reference assignment

### 7. Variables
- Primitive variables
- Reference variables
- Instance variables
- Static variables
- Local variables
- Default values
- Uninitialized arrays

### 8. Varargs
- Varargs syntax
- Varargs invocation
- Single-dimensional array vs varargs

### 9. Main Method
- `public static void main(String[] args)`
- Main method variations
- Java 7+ main-method enhancements

### 10. Command-Line Arguments
- Passing arguments
- Accessing arguments
- Practical examples

### 11. Java Coding Standards
- Class naming
- Interface naming
- Method naming
- Variable naming
- Constant naming
- JavaBean standards
- Getter/setter conventions
- Listener registration/unregistration

### 12. JVM Memory Areas — Introduction
- Stack
- Heap
- Method Area / Metaspace
- PC Register
- Native Method Stack

---

# Chapter 2 — Operators & Assignments

### Operators
- Increment and decrement
- Arithmetic operators
- String concatenation
- Relational operators
- Equality operators
- `instanceof`
- Bitwise operators
- Short-circuit operators
- Type-cast operator
- Assignment operators
- Conditional / ternary operator
- `new` operator
- `[]` operator

### Important Concepts
- Operator precedence
- Operand evaluation order
- `new` vs `newInstance()`
- `instanceof` vs `isInstance()`
- `ClassNotFoundException` vs `NoClassDefFoundError`

---

# Chapter 3 — Flow Control

### Selection
- `if`
- `if-else`
- Nested `if`
- `switch`
- Case rules
- Fall-through
- `default`

### Iterative
- `while`
- `do-while`
- `for`
- `for-each`
- Initialization section
- Conditional check
- Increment/decrement section
- Unreachable statements

### Iterators
- `Iterable`
- `Iterator`
- `Iterable` vs `Iterator`

### Transfer
- `break`
- `continue`
- Labeled `break`
- Labeled `continue`
- `do-while` + `continue` edge cases

---

# Chapter 4 — Declarations & Access Modifiers

### Modifiers
- `public`
- `protected`
- default/package-private
- `private`

### Declarations
- Class declaration
- Interface declaration
- Method declaration
- Variable declaration
- Constructor declaration
- Local vs instance vs static scope

### Interview Focus
- Access modifier visibility
- Package access
- Inheritance + `protected`
- Top-level class restrictions

---

# Chapter 5 — OOPs

### Class & Object
- Class
- Object
- State and behavior
- Object creation

### Encapsulation
- Data hiding
- Getters/setters
- Immutability connection

### Inheritance
- Single inheritance
- Multilevel inheritance
- Hierarchical inheritance
- Why Java does not support multiple class inheritance
- `super`
- Constructor chaining

### Polymorphism
- Compile-time polymorphism
- Method overloading
- Runtime polymorphism
- Method overriding
- Dynamic method dispatch

### Abstraction
- Abstract class
- Abstract method
- Interface
- Abstraction use cases

### Relationships
- Association
- Aggregation
- Composition
- Dependency

### Coupling & Cohesion
- Tight coupling
- Loose coupling
- High cohesion

### Must-Know Comparisons
- Abstract class vs interface
- Overloading vs overriding
- Aggregation vs composition
- Inheritance vs composition
- Encapsulation vs abstraction

---

# Chapter 6 — Exception Handling

### Fundamentals
- What is an exception?
- Exception hierarchy
- `Throwable`
- `Error`
- `Exception`
- `RuntimeException`

### Types
- Checked exceptions
- Unchecked exceptions
- Custom exceptions

### Keywords
- `try`
- `catch`
- `finally`
- `throw`
- `throws`

### Important Topics
- Multiple catch
- Nested try/catch
- Catch ordering
- Exception propagation
- Rethrowing exceptions
- `finally` behavior
- Try-with-resources
- Suppressed exceptions
- Custom checked vs unchecked exceptions

### Interview Comparisons
- `throw` vs `throws`
- Checked vs unchecked exception
- `final` vs `finally` vs `finalize`
- Error vs Exception

---

# Chapter 7 — Multithreading Fundamentals

> **Status:** 🚧 In Progress  
> **Goal:** Build strong Core Java concurrency fundamentals before moving to executors, concurrent collections, locks and `CompletableFuture`.

## 📋 Topic / Subtopic Tracker

| # | Topic / Subtopic | Status | Link |
|---:|---|---|---|
| 7.1 | Process vs Thread | ✅ Completed | [Open](07-Multithreading-Fundamentals/01-Process-vs-Thread/README.md) |
| 7.2 | Thread Creation — `Thread` class | ⏳ Pending | — |
| 7.3 | Thread Creation — `Runnable` | ⏳ Pending | — |
| 7.4 | Thread Lifecycle | ⏳ Pending | — |
| 7.5 | `Thread.State` and `RUNNABLE` | ⏳ Pending | — |
| 7.6 | Thread Naming & Basic Thread APIs | ⏳ Pending | — |
| 7.7 | `start()` vs `run()` | ⏳ Pending | — |
| 7.8 | `sleep()` | ⏳ Pending | — |
| 7.9 | `join()` | ⏳ Pending | — |
| 7.10 | `yield()` | ⏳ Pending | — |
| 7.11 | Race Condition | ⏳ Pending | — |
| 7.12 | Critical Section | ⏳ Pending | — |
| 7.13 | `synchronized` Method | ⏳ Pending | — |
| 7.14 | `synchronized` Block | ⏳ Pending | — |
| 7.15 | Intrinsic Monitor / Object Lock | ⏳ Pending | — |
| 7.16 | Class Lock vs Object Lock | ⏳ Pending | — |
| 7.17 | Reentrancy of `synchronized` | ⏳ Pending | — |
| 7.18 | Thread Safety | ⏳ Pending | — |
| 7.19 | Atomicity vs Visibility vs Ordering | ⏳ Pending | — |
| 7.20 | Happens-Before Relationship | ⏳ Pending | — |
| 7.21 | `volatile` Fundamentals | ⏳ Pending | — |
| 7.22 | `volatile` vs `synchronized` | ⏳ Pending | — |
| 7.23 | `wait()` | ⏳ Pending | — |
| 7.24 | `notify()` | ⏳ Pending | — |
| 7.25 | `notifyAll()` | ⏳ Pending | — |
| 7.26 | Monitor Ownership with `wait/notify` | ⏳ Pending | — |
| 7.27 | `wait()` vs `sleep()` | ⏳ Pending | — |
| 7.28 | Thread Interruption | ⏳ Pending | — |
| 7.29 | `interrupt()` | ⏳ Pending | — |
| 7.30 | Deadlock | ⏳ Pending | — |
| 7.31 | Starvation | ⏳ Pending | — |
| 7.32 | Livelock | ⏳ Pending | — |
| 7.33 | Thread Communication Patterns | ⏳ Pending | — |
| 7.34 | Common Thread-Safety Strategies | ⏳ Pending | — |
| 7.35 | Multithreading Interview Scenarios | ⏳ Pending | — |
| 7.36 | Multithreading Quick Revision | ⏳ Pending | — |
| 7.37 | Multithreading Final Assessment | ⏳ Pending | — |

### Completion Rule

A Chapter 7 topic is marked **✅ Completed** only after:

```text
Concept
+ Internal Working
+ Example
+ Code
+ Common Mistakes
+ Interview Questions
+ Quick Revision
```

---

# Chapter 8 — Multithreading Enhancements / Concurrency Utilities

### Executor Framework
- `Executor`
- `ExecutorService`
- `ScheduledExecutorService`
- Thread pools
- `Callable`
- `Future`

### Concurrent Collections
- `ConcurrentHashMap`
- `CopyOnWriteArrayList`
- Blocking queues

### Synchronizers
- `CountDownLatch`
- `CyclicBarrier`
- `Semaphore`
- `Phaser`
- `Exchanger`

### Atomic & Lock APIs
- `AtomicInteger`
- Atomic references
- `Lock`
- `ReentrantLock`
- `ReadWriteLock`
- `StampedLock`
- `Condition`

### Modern Concurrency
- `CompletableFuture`
- Async pipelines
- Composition vs blocking
- Exception handling in async flows
- Fork/Join Framework
- Parallel streams

---

# Chapter 9 — Inner Classes

- Member inner class
- Static nested class
- Local inner class
- Anonymous inner class
- Access rules
- Capturing local variables
- Inner class vs nested class
- Use cases

---

# Chapter 10 — `java.lang` Package

### Core Classes
- `Object`
- `String`
- `StringBuilder`
- `StringBuffer`
- Wrapper classes
- `System`
- `Math`
- `Class`
- `Enum`

### High-Priority Topics
- `equals()` vs `==`
- `hashCode()` contract
- String pool
- Immutability
- Wrapper caching / autoboxing
- `String` vs `StringBuilder` vs `StringBuffer`

---

# Chapter 11 — File I/O

### Streams
- Byte streams
- Character streams
- `InputStream`
- `OutputStream`
- `Reader`
- `Writer`

### File APIs
- `File`
- `FileInputStream`
- `FileOutputStream`
- `BufferedReader`
- `BufferedWriter`

### Modern NIO
- `Path`
- `Paths`
- `Files`
- `Channels`
- `Buffers`
- Directory traversal

---

# Chapter 12 — Serialization

- Serialization
- Deserialization
- `Serializable`
- `transient`
- `static` vs `transient`
- `transient` vs `final`
- Object graph
- Custom serialization
- Serialization with inheritance
- `Externalizable`
- Serialization vs Externalization
- `serialVersionUID`
- Serialization compatibility

---

# Chapter 13 — Regular Expressions

- Regex basics
- Character classes
- Quantifiers
- Groups
- Alternation
- Anchors
- `Pattern`
- `Matcher`
- `matches()` vs `find()` vs `lookingAt()`
- Common validation patterns

---

# Chapter 14 — Collections Framework

### Collection Hierarchy
- `Collection`
- `List`
- `Set`
- `Queue`
- `Deque`
- `Map` hierarchy

### List
- `ArrayList`
- `LinkedList`
- `Vector`
- `Stack`

### Set
- `HashSet`
- `LinkedHashSet`
- `TreeSet`

### Map
- `HashMap`
- `LinkedHashMap`
- `TreeMap`
- `Hashtable`
- `ConcurrentHashMap`

### Queue / Deque
- `PriorityQueue`
- `ArrayDeque`

### Must-Know Internals
- HashMap internal working
- Hash collision
- Java 8 treeification
- Load factor
- Resize
- Fail-fast vs weakly consistent iteration
- `equals()` + `hashCode()` contract
- Comparable vs Comparator

### Complexity & Selection
- ArrayList vs LinkedList
- HashMap vs TreeMap
- HashSet vs TreeSet
- HashMap vs ConcurrentHashMap
- PriorityQueue use cases

---

# Chapter 15 — Generics

- Generic introduction
- Type safety
- Generic classes
- Generic methods
- Generic interfaces
- Bounded types
- Upper bounds `extends`
- Lower bounds `super`
- Wildcard `?`
- PECS
- Type erasure
- Raw types
- Generic arrays limitations
- Generic varargs
- Interoperability with non-generic code

---

# Chapter 16 — Garbage Collection

### Basics
- What is garbage collection?
- Reachability
- Strong references
- Eligibility for GC

### JVM Memory
- Heap generations
- Young generation
- Old generation
- Metaspace

### GC Concepts
- Minor GC
- Major/old-generation collection
- Full GC
- Stop-the-world
- GC roots
- Object promotion
- Memory leaks in managed languages

### Modern GC Overview
- Serial GC
- Parallel GC
- G1 GC
- ZGC
- Shenandoah

### Interview Focus
- `System.gc()` hint
- Finalization deprecation/removal context
- Memory leak vs memory overflow
- Heap dump basics

---

# Chapter 17 — Enum

- Enum basics
- Enum constants
- Enum fields
- Enum constructors
- Enum methods
- `values()`
- `valueOf()`
- Enum in switch
- EnumSet
- EnumMap
- Enum singleton pattern

---

# Chapter 18 — Internationalization (i18n)

- Locale
- ResourceBundle
- Message formatting
- Number formatting
- Date/time formatting
- Currency formatting
- Unicode basics

---

# Chapter 19 — Development / Java Platform Essentials

- Packages
- Imports
- Compilation
- Classpath
- Module-path basics
- JARs
- Dependency management concepts
- Build lifecycle basics
- IDE vs compiler behavior
- Coding/debugging workflow

---

# Chapter 20 — Assertions

- What are assertions?
- `assert` syntax
- Enabling/disabling assertions
- Assertions vs exceptions
- When not to use assertions

---

# Chapter 21 — JVM Architecture

### JVM Components
- Class Loader Subsystem
- Runtime Data Areas
- Execution Engine
- Interpreter
- JIT Compiler
- Garbage Collector
- JNI

### Class Loading
- Loading
- Linking
- Verification
- Preparation
- Resolution
- Initialization

### Runtime Memory
- Heap
- Stack
- PC register
- Native method stack
- Metaspace

### Interview Focus
- JVM vs JRE vs JDK
- ClassLoader hierarchy
- Parent delegation
- JIT compilation
- StackOverflowError
- OutOfMemoryError

---

# Chapter 22 — Java 8+ New Features

### Java 8
- Lambda expressions
- Functional interfaces
- Method references
- Stream API
- `Optional`
- Default methods
- Static interface methods
- New Date/Time API

### Stream API
- `filter`
- `map`
- `flatMap`
- `sorted`
- `distinct`
- `limit`
- `skip`
- `reduce`
- `collect`
- `groupingBy`
- `partitioningBy`
- Parallel streams

### Java 9+
- Private interface methods
- Factory methods for collections
- `Optional` enhancements
- Reactive Streams interfaces overview

### Java 10+
- `var`

### Java 11+
- String convenience methods
- HTTP Client overview

### Java 14+
- Switch expressions

### Java 15+
- Text blocks

### Java 16+
- Pattern matching for `instanceof`
- Records

### Java 17+
- Sealed classes
- Strong encapsulation context

### Modern Java
- Pattern matching evolution
- Record patterns overview
- Modern switch pattern matching
- Virtual threads overview (Java 21+)
- Structured concurrency concepts

---

# Chapter 23 — String

### Fundamentals
- String immutability
- String pool
- String literals vs `new String()`
- `==` vs `equals()`
- `intern()`

### String Manipulation
- `substring`
- `charAt`
- `indexOf`
- `contains`
- `split`
- `replace`
- `trim`
- `strip`
- `concat`

### StringBuilder / StringBuffer
- Mutable strings
- Performance
- Thread-safety difference

### Interview Focus
- Why String is immutable
- String pool internals
- Why String is a good HashMap key
- String concatenation internals
- `StringBuilder` vs `StringBuffer`

---

# Interview Priority Map

## 🔥 Priority 1 — Must Master
- OOPs
- String / Immutability
- Collections
- HashMap internals
- `equals()` / `hashCode()`
- Exception Handling
- Multithreading
- Executor Framework
- Java 8 Stream API
- Lambda / Functional Interface
- JVM Architecture
- Garbage Collection

## 🔥 Priority 2 — Strong Understanding
- Generics
- Inner Classes
- Serialization
- File I/O
- Regular Expressions
- Enum
- Internationalization
- Assertions

## 📌 Standard Topic Completion Format

Every topic should be completed in this order:

```text
1. Concept
2. Why it exists
3. Internal Working
4. Example
5. Java Code
6. Common Mistakes
7. Interview Questions
8. 2-Minute Interview Answer
9. Quick Revision
10. Final Assessment
```

### Current Progress

**Chapter 7 → 7.1 Process vs Thread → ✅ Completed**

**Next → 7.2 Thread Creation — `Thread` class**