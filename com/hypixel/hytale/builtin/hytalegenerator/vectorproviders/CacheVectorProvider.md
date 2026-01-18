---
description: Architectural reference for CacheVectorProvider
---

# CacheVectorProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.vectorproviders
**Type:** Transient Decorator

## Definition
```java
// Signature
public class CacheVectorProvider extends VectorProvider {
```

## Architecture & Concepts
The CacheVectorProvider is a performance-optimization component that implements the Decorator pattern. It wraps another, potentially computationally expensive, VectorProvider to provide a layer of memoization.

Its primary function is to cache the last computed result for a given spatial position. This is particularly effective in procedural generation scenarios where the same location may be queried multiple times in succession by the same processing thread.

The key architectural feature is its thread-aware caching mechanism. It does not use a single, global cache, which would require expensive synchronization. Instead, it maintains a distinct cache for each worker thread, indexed by the `workerId` from the `VectorProvider.Context`. This design completely avoids lock contention and ensures high-throughput, thread-safe operation by partitioning data per thread.

## Lifecycle & Ownership
- **Creation:** Instantiated during the construction of a world generation pipeline. It is configured with the target VectorProvider to wrap and the total number of worker threads that will participate in the generation process.
- **Scope:** The instance persists for the lifetime of the generator configuration it is a part of. It is stateful, but its state is tied to a specific generation session.
- **Destruction:** The object and its internal thread-local caches are eligible for garbage collection once the parent generator is no longer referenced. It does not manage any unmanaged resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The CacheVectorProvider is highly stateful. Its internal state consists of a collection of `Cache` objects, one for each worker thread, managed by a `WorkerIndexer.Data` instance. Each `Cache` object stores the last processed position and its corresponding result vector.
- **Thread Safety:** This class is thread-safe by design. Thread safety is achieved through data partitioning rather than explicit locking. Each worker thread accesses its own exclusive `Cache` instance via its unique `workerId`. This architecture guarantees that no two threads will ever concurrently modify the same cache entry, eliminating the possibility of data races.

## API Surface
The public contract is minimal, adhering to the `VectorProvider` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | Vector3d | O(1) or O(N) | Returns a vector for the context's position. Complexity is O(1) on a cache hit. On a miss, complexity is O(N), where N is the complexity of the wrapped provider's process call. |

## Integration Patterns

### Standard Usage
The CacheVectorProvider is used to wrap expensive providers, such as those involving complex noise sampling or data lookups, within a multi-threaded generation system.

```java
// Assume 'expensiveNoiseProvider' is a costly VectorProvider to evaluate.
// The 'workerCount' must match the number of threads in the generator's pool.
int workerCount = 16;
VectorProvider baseProvider = new ExpensiveNoiseProvider(...);

// Wrap the base provider to add caching capabilities.
VectorProvider cachedProvider = new CacheVectorProvider(baseProvider, workerCount);

// In a worker thread, use the cachedProvider. The first call for a new
// position will be slow, but subsequent calls by the same thread for the
// same position will be near-instantaneous.
Vector3d result = cachedProvider.process(workerContext);
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Thread Count:** Initializing with a `threadCount` that is less than the highest `workerId` passed to the `process` method will result in an `ArrayIndexOutOfBoundsException`.
- **Wrapping Inexpensive Providers:** Wrapping a simple, stateless, or computationally cheap provider (e.g., a `ConstantVectorProvider`) adds unnecessary memory and processing overhead for cache management with no performance benefit.
- **Assuming Cross-Thread Caching:** Do not expect a value cached by Worker Thread 1 to be available to Worker Thread 2. The cache is strictly local to each thread.

## Data Pipeline
The component acts as a conditional passthrough in the data pipeline. It intercepts the request and either serves it directly from its local cache or forwards it to the decorated provider.

> Flow:
> VectorProvider.Context -> **CacheVectorProvider** (Cache Check) -> [Optional] Wrapped VectorProvider -> Vector3d

