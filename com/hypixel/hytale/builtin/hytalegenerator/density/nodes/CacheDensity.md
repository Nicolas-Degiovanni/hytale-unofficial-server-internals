---
description: Architectural reference for CacheDensity
---

# CacheDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Wrapper

## Definition
```java
// Signature
public class CacheDensity extends Density {
```

## Architecture & Concepts
The CacheDensity class is a performance-optimization component within the procedural world generation pipeline. It functions as a Decorator, wrapping another, more computationally expensive, Density node. Its sole purpose is to memoize, or cache, the result of its input node for a specific 3D coordinate.

This node is critically designed for a multi-threaded generation environment. Instead of using a shared, locked cache which would create contention, it leverages a thread-local storage pattern via the WorkerIndexer utility. Each generator worker thread is allocated its own private cache. This lock-free design ensures high-throughput processing by eliminating synchronization overhead between threads.

The caching strategy is simple and effective for its intended use case: it stores only the result of the single most recent coordinate processed by a given thread. This is highly effective when a complex Density calculation is requested multiple times for the exact same coordinate within a localized generation task, a common pattern in procedural generation algorithms.

### Lifecycle & Ownership
- **Creation:** Instantiated by the world generation graph composer. It is not a standalone service but a node within a larger, composite Density graph. It is constructed with a reference to the input Density it will cache and the total number of worker threads in the generator's thread pool.
- **Scope:** The lifetime of a CacheDensity instance is tied to the specific world generation graph it is part of. These graphs are typically transient, existing only for the duration of a single, large-scale generation operation (e.g., generating a world region).
- **Destruction:** The object is marked for garbage collection when the parent generation graph is no longer referenced. It holds no persistent state and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is highly mutable by design. Its internal state consists of an array of private Cache objects, indexed by worker thread ID. Each Cache object stores a Vector3d position and the corresponding double value. This state is updated on every cache miss.
- **Thread Safety:** The class is thread-safe. Concurrency is managed by data partitioning rather than locking. Each worker thread operates on an independent index in the `threadData` array, as determined by the `context.workerId`. This architecture guarantees that no two threads will ever write to the same Cache object, completely avoiding race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) or O(N) | Returns the density value for the given context. Complexity is O(1) on a cache hit. On a miss, complexity is O(N), where N is the complexity of the wrapped input node's process method. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the underlying Density node being cached. Throws AssertionError if the input array is empty or contains null. |

## Integration Patterns

### Standard Usage
CacheDensity should be used to wrap computationally expensive branches of a Density graph. The `threadCount` provided during construction **must** match the number of worker threads used by the world generator to prevent runtime exceptions.

```java
// Assume 'complexDensityNode' is a costly operation (e.g., multi-octave noise)
Density complexDensityNode = new PerlinNoise(...);

// The generator's thread pool size is known
int workerThreadCount = worldGenerator.getThreadCount();

// Wrap the complex node in a CacheDensity to optimize repeated lookups
Density cachedNode = new CacheDensity(complexDensityNode, workerThreadCount);

// Use the cached node within the broader generation graph
double value = cachedNode.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Thread Count:** Initializing with a `threadCount` that does not match the active number of generator worker threads will cause an `ArrayIndexOutOfBoundsException` when a worker with an out-of-bounds ID attempts to access its cache.
- **Wrapping Inexpensive Nodes:** Do not wrap simple or computationally cheap Density nodes, such as a ConstantDensity. The overhead of the position check and branching logic within CacheDensity will be greater than the cost of re-computing the input, resulting in a net performance loss.
- **Wrapping Non-Deterministic Nodes:** The cache assumes that for a given position, the input node will always produce the same result. Wrapping a node with random or time-dependent behavior will produce incorrect and inconsistent generation results.

## Data Pipeline
The data flow for a single `process` call is determined by the cache state for the calling thread.

> Flow on Cache Miss:
> Density.Context -> **CacheDensity.process()** -> Check Thread-Local Cache -> [Miss] -> Invoke `input.process()` -> Update Thread-Local Cache with new position and value -> Return new value

> Flow on Cache Hit:
> Density.Context -> **CacheDensity.process()** -> Check Thread-Local Cache -> [Hit] -> Return stored `cache.value` directly

