---
description: Architectural reference for HytaleForkJoinThreadFactory
---

# HytaleForkJoinThreadFactory

**Package:** com.hypixel.hytale.common.thread
**Type:** Utility

## Definition
```java
// Signature
public class HytaleForkJoinThreadFactory implements ForkJoinWorkerThreadFactory {
```

## Architecture & Concepts
The HytaleForkJoinThreadFactory is a specialized factory class that integrates Hytale's diagnostic and metrics systems into Java's core concurrency framework. Its sole purpose is to customize the creation of worker threads for a ForkJoinPool.

Standard Java ForkJoinPools create generic worker threads. This factory replaces that default behavior, instead producing HytaleForkJoinThreadFactory.WorkerThread instances. Each custom worker thread, upon its creation, captures the stack trace of the thread that initiated its construction.

This mechanism is critical for performance profiling and debugging in a highly parallel environment. It allows developers to answer the question: "Why was this thread pool created, and what code path led to its existence?". This information is exposed via the InitStackThread interface and consumed by higher-level engine monitoring services. This class acts as a crucial injection point for engine observability into a standard Java utility.

### Lifecycle & Ownership
- **Creation:** An instance of HytaleForkJoinThreadFactory is created directly during the configuration and instantiation of a new ForkJoinPool. It is passed as an argument to the ForkJoinPool constructor.
- **Scope:** The factory's lifecycle is bound to the ForkJoinPool that owns it. It persists as long as the pool is active.
- **Destruction:** The factory instance is eligible for garbage collection once the owning ForkJoinPool is shut down and all references to it are released. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The HytaleForkJoinThreadFactory is stateless and immutable. It contains no instance fields and its behavior is consistent across all calls. The threads it creates, however, contain a final field, initStack, which is captured once at construction and is immutable thereafter.
- **Thread Safety:** This class is inherently thread-safe. The newThread method is a pure factory function with no side effects. It can be safely invoked by the ForkJoinPool's internal management logic from any thread without synchronization.

## API Surface
The public API is minimal, adhering strictly to the ForkJoinWorkerThreadFactory interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newThread(ForkJoinPool pool) | ForkJoinWorkerThread | O(N) | Creates a new WorkerThread. Called exclusively by a ForkJoinPool. The complexity is proportional to the depth (N) of the creator's call stack due to the getStackTrace call. |

**Warning:** The call to capture the stack trace is not a zero-cost operation. This factory is intended for pools where the diagnostic benefit of knowing the thread's origin outweighs the minor overhead during thread creation.

## Integration Patterns

### Standard Usage
This factory should only be used to construct a ForkJoinPool. It provides the pool with Hytale-specific, diagnosable worker threads.

```java
// Correctly create a ForkJoinPool with enhanced diagnostics
int parallelism = Runtime.getRuntime().availableProcessors();
ForkJoinWorkerThreadFactory factory = new HytaleForkJoinThreadFactory();
UncaughtExceptionHandler handler = new HytaleUncaughtExceptionHandler();

ForkJoinPool taskPool = new ForkJoinPool(parallelism, factory, handler, false);
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the newThread method directly. The ForkJoinPool manages the thread lifecycle. Manual invocation will create a thread that is not properly managed by the pool, leading to resource leaks and unpredictable behavior.
- **Use with other Executors:** This factory is specifically designed for the ForkJoinPool. Attempting to use it with other ExecutorService implementations will result in a compilation error or runtime class cast exceptions.

## Data Pipeline
This class does not process data in a traditional pipeline. Instead, it participates in a "creation and diagnostics" flow.

> Flow:
> ForkJoinPool requires a new worker -> Invokes **HytaleForkJoinThreadFactory**.newThread -> Factory captures creator's stack trace -> A new **WorkerThread** is instantiated with the stack trace -> The thread is returned to and managed by the ForkJoinPool -> At a later time, a Metrics or Debugging System queries the thread -> The thread provides its initial stack trace via the InitStackThread interface.

