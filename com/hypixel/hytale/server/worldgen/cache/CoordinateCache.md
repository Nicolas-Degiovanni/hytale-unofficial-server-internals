---
description: Architectural reference for CoordinateCache
---

# CoordinateCache<T>

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Transient Component (Abstract Base Class)

## Definition
```java
// Signature
public abstract class CoordinateCache<T> {
```

## Architecture & Concepts
The CoordinateCache is an abstract, high-performance caching layer designed specifically for the server-side world generation pipeline. Its primary function is to memoize the results of expensive computations tied to specific world coordinates and a world seed, drastically reducing redundant calculations during chunk generation.

This class is not a generic cache; it is purpose-built for a high-throughput, low-allocation environment. The architecture is centered on two key optimizations:

1.  **Key Object Pooling:** To combat garbage collection pressure in the performance-critical world generation loop, the cache does not create new key objects for every lookup. It maintains an internal ObjectPool of CoordinateKey instances. Lookups reuse objects from this pool, significantly reducing memory churn.
2.  **Time and Size-Based Eviction:** The cache is backed by a SizedTimeoutCache. This ensures that the memory footprint of the cache is bounded and that stale data is automatically evicted after a configurable duration. This makes it suitable for caching transient world generation data that is frequently accessed for a short period and then becomes irrelevant.

Subclasses must provide the core business logic by implementing the `compute` method, which defines how to generate a value when a cache miss occurs. They must also implement `onRemoval` for any necessary cleanup when a value is evicted from the cache.

## Lifecycle & Ownership
-   **Creation:** An instance of a concrete CoordinateCache subclass is created and configured by a higher-level world generation service. The constructor requires `maxSize` and `expireAfterSeconds`, indicating its lifecycle and memory constraints are defined by its owner.
-   **Scope:** The cache's lifetime is tightly coupled to its owning service, typically a specific world or dimension generator. It persists as long as that world is active and its data is relevant. It is not a global or static cache.
-   **Destruction:** The object is eligible for garbage collection when its owning world generation service is destroyed. The underlying SizedTimeoutCache ensures that upon item eviction, the `onRemoval` callback is fired and the associated CoordinateKey is returned to the internal object pool for reuse.

## Internal State & Concurrency
-   **State:** This class is stateful and mutable. Its primary state is stored within the composed SizedTimeoutCache and the ObjectPool for keys. The contents of the cache change dynamically as items are added, accessed, and evicted.
-   **Thread Safety:** This component is designed for use in a multi-threaded environment, such as parallel chunk generation. Thread safety is provided by the underlying SizedTimeoutCache and ObjectPool implementations, which are expected to handle concurrent access, insertions, and removals. Callers from multiple threads can safely invoke the `get` method.

## API Surface
The public contract is minimal, focusing on data retrieval. The primary extension points are the protected abstract methods for subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed, int x, int y) | T | O(1) avg | Retrieves an entry from the cache. If the entry is not present, it triggers the `compute` method, caches the result, and returns it. A cache miss will have the complexity of the `compute` implementation. |
| localKey() | CoordinateKey | O(1) | *Protected.* Abstract method for subclasses to provide a thread-local key instance for lookups. |
| compute(int seed, int x, int z) | T | User-Defined | *Protected.* Abstract method defining the logic to generate a value on a cache miss. **Warning:** This method will block the calling thread and must be highly optimized. |
| onRemoval(T value) | void | User-Defined | *Protected.* Abstract callback invoked when an entry is evicted from the cache due to size or timeout. Used for resource cleanup. |

## Integration Patterns

### Standard Usage
A developer must extend CoordinateCache, implement the abstract methods, and then use the instance to fetch or generate data.

```java
// 1. Define a concrete cache implementation
public class BiomeCache extends CoordinateCache<Biome> {
    private final ThreadLocal<CoordinateKey> localKey = ThreadLocal.withInitial(CoordinateKey::new);

    public BiomeCache(int maxSize, long expireAfterSeconds) {
        super(maxSize, expireAfterSeconds);
    }

    @Override
    protected CoordinateKey localKey() {
        return this.localKey.get();
    }

    @Override
    protected Biome compute(int seed, int x, int z) {
        // Expensive biome generation logic here...
        return BiomeGenerator.calculateBiomeAt(seed, x, z);
    }

    @Override
    protected void onRemoval(Biome biome) {
        // No-op if Biome objects require no special cleanup
    }
}

// 2. Instantiate and use the cache in a service
BiomeCache biomeCache = new BiomeCache(8192, 300);
Biome currentBiome = biomeCache.get(world.getSeed(), player.getX(), player.getZ());
```

### Anti-Patterns (Do NOT do this)
-   **Slow Computation:** Implementing a `compute` method with high latency or blocking I/O will cause severe performance degradation, as it blocks the caller on a cache miss.
-   **Ignoring Thread Locality:** Failing to provide a thread-local CoordinateKey via the `localKey` method can introduce race conditions if a single key instance is shared across threads.
-   **Caching Mutable Objects:** Storing and returning mutable objects from the cache is dangerous. If multiple threads get a reference to the same cached object, they can cause race conditions by modifying its state concurrently without proper synchronization. Cache immutable objects or defensive copies where possible.

## Data Pipeline
The data flow for a `get` operation is a standard cache-aside pattern, optimized with object pooling.

> Flow:
> `get(seed, x, y)` -> Acquire thread-local `CoordinateKey` -> **SizedTimeoutCache Lookup**
>
> **On Cache Hit:**
> Return cached value `T` -> Release `CoordinateKey` to pool
>
> **On Cache Miss:**
> Invoke `compute(seed, x, z)` -> Generate new value `T` -> Store `T` in cache -> Return new value `T` -> Release `CoordinateKey` to pool

