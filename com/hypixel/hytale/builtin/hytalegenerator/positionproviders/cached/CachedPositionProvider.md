---
description: Architectural reference for CachedPositionProvider
---

# CachedPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders.cached
**Type:** Transient

## Definition
```java
// Signature
public class CachedPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The CachedPositionProvider is a high-performance decorator that wraps another PositionProvider to introduce a caching layer. Its primary function is to reduce the computational cost of position generation by memoizing results, which is critical in performance-sensitive systems like procedural world generation.

This class implements a spatial partitioning cache. It divides the world into a grid of discrete, cubic "sections". When a request for positions within a given volume is made, the provider calculates which sections intersect that volume. It then queries its internal cache for each of these sections.

-   **On a cache hit,** the pre-computed positions for that section are retrieved directly from memory.
-   **On a cache miss,** it delegates the generation request for that specific section to the underlying, wrapped PositionProvider. The result is then stored in the cache before being returned.

A key architectural feature is its support for multi-threaded environments. It maintains a separate, isolated cache for each worker thread, identified by a workerId. This design completely avoids lock contention and synchronization overhead, enabling highly parallel execution by ensuring that threads never compete for cache access. The cache itself employs a Least Recently Used (LRU) eviction policy to manage memory consumption.

## Lifecycle & Ownership
-   **Creation:** Instantiated manually by a higher-level system, typically a world generation orchestrator. It is constructed by wrapping an existing PositionProvider instance that requires performance optimization.
-   **Scope:** The lifetime of a CachedPositionProvider is tied to a specific generation task or pipeline. It is not a global singleton and holds state relevant only to its configured task.
-   **Destruction:** The object and its caches are eligible for garbage collection as soon as the orchestrator that created it releases its reference. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** This component is highly stateful and mutable. Its primary state is stored in the `threadData` field, which maps worker IDs to `CacheThreadMemory` instances. Each of these instances contains a map of section data and a doubly-linked list to manage the LRU eviction strategy. The state is continuously modified during calls to `positionsIn`.

-   **Thread Safety:** This class is thread-safe when used as intended within a parallel worker system. Safety is achieved through data partitioning, not locking. The `WorkerIndexer.Data` structure guarantees that each worker thread accesses a completely separate cache instance.

    **Warning:** While concurrent calls from different worker threads are safe, it is not safe to have multiple threads making calls using the same workerId. The system architecture assumes a one-to-one relationship between a thread and its assigned workerId for the duration of a task.

## API Surface
The public contract is inherited from the base PositionProvider. The constructor is the primary configuration point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CachedPositionProvider(...) | constructor | O(T) | Constructs the provider. Complexity is proportional to threadCount (T). |
| positionsIn(Context) | void | Amortized O(S) | Fulfills the position request. Complexity is amortized over many calls. S is the number of sections intersecting the query volume. A cache miss incurs the full cost of the decorated provider for the affected section. |

## Integration Patterns

### Standard Usage
The CachedPositionProvider is used to wrap an existing, computationally expensive provider. It is configured with parameters that tune its caching behavior, such as the section size, cache capacity, and the number of worker threads it must support.

```java
// 1. Define the base provider which may be slow
PositionProvider expensiveProvider = new NoiseBasedPositionProvider(...);

// 2. Define cache parameters
int sectionGridSize = 32; // Cache positions in 32x32x32 chunks
int sectionsToCachePerThread = 1024;
int worldgenThreadCount = 8;

// 3. Decorate the base provider with the caching layer
PositionProvider optimizedProvider = new CachedPositionProvider(
    expensiveProvider,
    sectionGridSize,
    sectionsToCachePerThread,
    true,
    worldgenThreadCount
);

// 4. Use the optimized provider within a worker thread context
// The context provides the query volume and the unique workerId
PositionProvider.Context workerContext = new PositionProvider.Context(min, max, consumer, world, workerId);
optimizedProvider.positionsIn(workerContext);
```

### Anti-Patterns (Do NOT do this)
-   **Incorrect Sizing:** Configuring an inappropriate `sectionSize`. A size that is too small leads to high overhead and cache fragmentation. A size that is too large causes the system to generate and store excessive data for small queries, a phenomenon known as over-fetching.
-   **Cache Thrashing:** Setting the `cacheSize` too low for the workload's access patterns. This will cause frequent evictions and re-computations, defeating the purpose of the cache and potentially performing worse than the undecorated provider.
-   **Wrapping a Fast Provider:** Do not wrap a provider that is already computationally trivial. The overhead of the caching logic itself (hashing, map lookups, bounds checking) can make the final result slower.

## Data Pipeline
The component intercepts a request for positional data, attempts to fulfill it from a cache, and delegates to a downstream provider on a cache miss.

> Flow (Cache Miss):
> Position Request -> **CachedPositionProvider** -> Section Calculation -> Cache Lookup (Miss) -> Delegate to decorated PositionProvider -> Generate Section Data -> Update Cache -> Filter Results -> Output Consumer

> Flow (Cache Hit):
> Position Request -> **CachedPositionProvider** -> Section Calculation -> Cache Lookup (Hit) -> Retrieve Section Data -> Filter Results -> Output Consumer

