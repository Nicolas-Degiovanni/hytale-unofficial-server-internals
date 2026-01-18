---
description: Architectural reference for MultiCacheDensity
---

# MultiCacheDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class MultiCacheDensity extends Density {
```

## Architecture & Concepts
MultiCacheDensity is a specialized node within the procedural world generation framework. It functions as a high-performance, thread-local memoization layer. Architecturally, it implements the **Decorator Pattern**, wrapping another Density node (the *input*) to cache its output values.

Its primary purpose is to optimize computationally expensive density graphs. By storing the results of its input node, it prevents redundant calculations for the same world position.

The "Multi" in its name refers to its core design principle: it maintains a separate, independent cache for each worker thread involved in world generation. This is achieved through the WorkerIndexer utility, which maps a worker ID to a dedicated cache instance. This thread-affine caching strategy completely avoids lock contention and synchronization overhead, making it exceptionally efficient in highly parallel generation tasks.

## Lifecycle & Ownership
-   **Creation:** Instantiated during the construction of a density graph. This typically occurs when a world generator's configuration is parsed and the corresponding node graph is assembled in memory. It is not managed by a dependency injection container.
-   **Scope:** The object's lifetime is bound to the density graph it belongs to. It persists as long as the graph is held in memory for a generation task.
-   **Destruction:** As a standard Java object with no native resources, it is eligible for garbage collection once the density graph is no longer referenced. No explicit cleanup methods are required.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. Its primary state is stored in the `threadData` field, which holds a collection of `Cache` objects—one for each worker thread. Each `Cache` object contains a fixed-size circular buffer of previously computed density values and their corresponding positions.
-   **Thread Safety:** This class is **not thread-safe** in a general-purpose context. It is specifically designed for a thread-affine concurrency model where each thread is assigned a unique `workerId`. All calls to the `process` method from a single thread must use the same `workerId`. Accessing the same cache instance (by using the same `workerId` from multiple threads simultaneously) will lead to data corruption and race conditions. The design deliberately trades general thread safety for maximum performance by eliminating locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(capacity) | Computes the density. First, it performs a linear scan of the thread-local cache. On a cache hit, it returns the stored value. On a miss, it calls `process` on its input, stores the result, and returns it. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the underlying `Density` node that this object caches. Throws AssertionError if the input array is empty or contains null. |

## Integration Patterns

### Standard Usage
MultiCacheDensity is used to wrap and optimize another, more expensive Density node within a larger graph.

```java
// Assume 'expensiveNoise' is a complex, slow Density node
Density expensiveNoise = new PerlinNoise(...);

// Wrap it with a cache for 16 threads, with 256 entries per thread
int threadCount = 16;
int cacheCapacity = 256;
Density cachedNoise = new MultiCacheDensity(expensiveNoise, threadCount, cacheCapacity);

// Use 'cachedNoise' in the rest of the density graph.
// The 'process' method will now be memoized per-thread.
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Worker IDs:** Never allow two concurrent threads to call `process` using the same `workerId`. This violates the core design assumption and will corrupt the cache state. Each worker thread must have a unique, consistent ID for the duration of its task.
-   **Incorrect Capacity:** Setting the `capacity` too low will result in cache thrashing, where useful values are evicted too quickly, negating the performance benefit. Setting it too high consumes excessive memory and can slow down the cache lookup, which is a linear scan.
-   **Dynamic Input Swapping:** Calling `setInputs` to change the underlying `Density` node while a world generation task is actively running across multiple threads is not safe and can lead to inconsistent results. Any structural changes to the graph should be performed between generation tasks.

## Data Pipeline
The data flow for a single `process` call is designed to prioritize fast cache lookups.

> Flow:
> Density.Context (position, workerId) -> **MultiCacheDensity** -> Select thread-local Cache via workerId -> Scan Cache for position -> (Cache Hit) -> Return stored value
>
> (Cache Miss) -> Call `input.process()` -> Store result in circular buffer -> Return new value

