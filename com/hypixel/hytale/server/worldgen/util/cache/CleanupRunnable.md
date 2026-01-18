---
description: Architectural reference for CleanupRunnable
---

# CleanupRunnable

**Package:** com.hypixel.hytale.server.worldgen.util.cache
**Type:** Transient Task

## Definition
```java
// Signature
public class CleanupRunnable<K, V> implements Runnable {
```

## Architecture & Concepts
The CleanupRunnable is a specialized, single-purpose task object designed to perform maintenance on a Cache instance. It serves as a critical component of the engine's asynchronous cache management system, specifically for garbage collection and eviction of stale entries.

Its architecture is defined by two key characteristics:
1.  **Runnable Interface:** By implementing Runnable, it conforms to the standard Java concurrency contract, allowing it to be submitted to an ExecutorService for background execution. This decouples the potentially expensive cache cleanup operation from performance-sensitive game loop or network threads.
2.  **WeakReference Strategy:** The class holds a WeakReference to its target Cache, not a direct, strong reference. This is a crucial design choice to prevent memory leaks. If the Cache is no longer strongly referenced elsewhere in the application and becomes eligible for garbage collection, this Runnable will not prevent it. The weak link is broken, and the cleanup task gracefully becomes a no-operation.

This class embodies the "Janitor" pattern, where a background process is responsible for periodically tidying up resources owned by a primary component.

### Lifecycle & Ownership
-   **Creation:** An instance of CleanupRunnable is typically created by the Cache it is intended to service. The Cache, upon its own initialization, will instantiate this runnable and schedule it with a shared, system-wide ScheduledExecutorService.
-   **Scope:** The object's lifecycle is managed by the executor service. It persists only as long as it is scheduled for execution. Once the run method completes, the executor releases its reference, making the object eligible for garbage collection.
-   **Destruction:** The object is destroyed by the Java Garbage Collector after its execution is complete and it is no longer referenced by the scheduling service.

## Internal State & Concurrency
-   **State:** The internal state is minimal and effectively immutable. It consists of a single final field, a WeakReference to the target Cache. The Runnable itself does not manage or mutate any state; it only triggers an operation on its target.
-   **Thread Safety:** This class is inherently thread-safe. Its state is established at construction and never changes. The run method is designed to be executed by a single thread from an executor pool.

    **WARNING:** While the Runnable is thread-safe, it provides no guarantees about the thread safety of the target Cache. It is assumed that the `Cache.cleanup()` method is itself designed to handle concurrent access safely.

## API Surface
The public contract is defined entirely by the Runnable interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run() | void | O(N) | Executes the cleanup logic on the target Cache. The complexity is dependent on the implementation of `Cache.cleanup()`. If the target Cache has been garbage collected, this method is a no-op with O(1) complexity. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is an internal component of the caching framework. A Cache implementation that requires periodic cleanup would use it as follows, typically in its constructor.

```java
// Example from a hypothetical Cache class constructor
// A central scheduler is retrieved from a service context.
ScheduledExecutorService scheduler = context.getScheduler();

// A WeakReference to 'this' cache is created.
WeakReference<Cache<K, V>> weakSelf = new WeakReference<>(this);

// The cleanup task is created and scheduled to run periodically.
scheduler.scheduleAtFixedRate(
    new CleanupRunnable<>(weakSelf),
    INITIAL_DELAY_SECONDS,
    PERIOD_SECONDS,
    TimeUnit.SECONDS
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call `run()` directly from a critical application thread. Doing so would block the thread and negate the entire purpose of the asynchronous design.
-   **Using Strong References:** Modifying this class to use a strong reference to the Cache would create a memory leak. The Cache would never be garbage collected as long as the scheduled task exists in the executor, creating a circular dependency.

## Data Pipeline
This class does not participate in a data processing pipeline. Instead, it acts as a trigger in a control flow for resource management.

> Flow:
> ScheduledExecutorService Thread -> **CleanupRunnable.run()** -> Cache.cleanup() -> Stale entries evicted from Cache memory

