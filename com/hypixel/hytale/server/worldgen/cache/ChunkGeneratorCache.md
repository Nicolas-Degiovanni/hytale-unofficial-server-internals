---
description: Architectural reference for ChunkGeneratorCache
---

# ChunkGeneratorCache

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Stateful Component

## Definition
```java
// Signature
public class ChunkGeneratorCache {
```

## Architecture & Concepts
The ChunkGeneratorCache is a high-performance, specialized caching layer for memoizing expensive world generation calculations. It sits at the core of the world generation pipeline, acting as a fast-lookup service for the ChunkGenerator to prevent redundant computations for biome, height, and noise data at specific world coordinates.

This system is not a simple key-value store; it is an intelligent, read-through cache. When data for a coordinate is requested, the cache first checks for its existence. On a cache miss, it uses strategy functions, provided during its construction, to compute the required data, stores the result for subsequent requests, and then returns it. This pattern centralizes caching logic and decouples it from the core generation algorithms.

Key architectural features include:
*   **Lazy Evaluation:** The primary cached object, CoreDataCacheEntry, only computes its most basic data on a cache miss. Secondary data, such as height or biome counts, are only computed and populated on-demand when their specific accessor methods are first called for that entry.
*   **Performance Optimization:** The implementation is heavily optimized for high-throughput, concurrent environments. It uses an internal object pool for its cache keys to minimize garbage collector pressure and leverages a thread-local key for lookups to avoid object allocation on the hot path.
*   **Configurable Eviction:** The cache is bounded by both size and time. Entries are automatically evicted after a configured duration or when the cache exceeds its maximum size, ensuring a predictable memory footprint.

## Lifecycle & Ownership
- **Creation:** A ChunkGeneratorCache is instantiated by a ChunkGenerator during its own initialization. It is not a global singleton. Each generator instance owns a dedicated cache, configured with computation functions specific to that generator's world seed and parameters.
- **Scope:** The cache's lifecycle is strictly bound to its parent ChunkGenerator. It lives for the duration of the world generation process managed by that generator.
- **Destruction:** The object is eligible for garbage collection when its owning ChunkGenerator is de-referenced and destroyed. It has no explicit cleanup or shutdown method. Internal entries are managed by the time-based eviction policy.

## Internal State & Concurrency
- **State:** The ChunkGeneratorCache is highly mutable. Its core state is the internal ConcurrentSizedTimeoutCache, which holds a map of CoordinateKey objects to CoreDataCacheEntry objects. This state is volatile, with entries constantly being added, lazily updated, and evicted.
- **Thread Safety:** This class is thread-safe and designed for high-concurrency access from multiple world generation worker threads. Thread safety is primarily delegated to the underlying ConcurrentSizedTimeoutCache. The concurrency level of the internal cache is explicitly tuned based on the ChunkGenerator's worker pool size, and the use of a thread-local key for lookups (`localKey`) prevents contention.

## API Surface
The public API provides access to various computed world generation data points, abstracting away the underlying caching and lazy evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, z) | CoreDataCacheEntry | Amortized O(1) | Retrieves the master cache entry for a coordinate, triggering a full computation on a cache miss. |
| getZoneBiomeResult(seed, x, z) | ZoneBiomeResult | Amortized O(1) | Returns the pre-computed biome result for a coordinate. |
| getBiomeCountResult(seed, x, z) | InterpolatedBiomeCountList | Amortized O(1) | Returns biome count data, triggering its computation if not already present in the entry. |
| getHeight(seed, x, z) | int | Amortized O(1) | Returns the terrain height, triggering its computation if not already present in the entry. |
| putHeight(seed, x, z, height) | void | Amortized O(1) | Directly injects a height value into an existing or new cache entry. |

**Warning:** The complexity of all `get` methods is amortized O(1). A cache miss will incur the full cost of the underlying computation functions, which can be significant.

## Integration Patterns

### Standard Usage
The cache is intended to be used exclusively by a ChunkGenerator. The generator provides its own terrain calculation methods as function references during the cache's construction.

```java
// Within a ChunkGenerator or similar context
// The functions (this::computeBiomes, etc.) are expensive operations
ChunkGeneratorCache worldCache = new ChunkGeneratorCache(
    this::computeZoneBiomes,
    this::computeBiomeCounts,
    this::computeHeight,
    this::computeHeightNoise,
    16384, // maxSize
    120    // expireAfterSeconds
);

// Later, during chunk processing...
int height = worldCache.getHeight(worldSeed, chunkX, chunkZ);
ZoneBiomeResult biomes = worldCache.getZoneBiomeResult(worldSeed, chunkX, chunkZ);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create a ChunkGeneratorCache outside of a world generation context. Its lifecycle and configuration are tightly coupled to a parent ChunkGenerator.
- **State Sharing:** Never share a single cache instance between different ChunkGenerator instances, especially those for different worlds or seeds. This will lead to data corruption and incorrect world generation.
- **Improper Configuration:** Using a `maxSize` that is too small for the workload will cause constant cache eviction and re-computation, known as thrashing, negating the performance benefits of the cache.

## Data Pipeline
The ChunkGeneratorCache acts as a service node in the data flow rather than a linear pipeline stage. Its primary role is to short-circuit expensive computations.

> Flow:
> 1. ChunkGenerator requests data for a coordinate (e.g., height).
> 2. The request enters **ChunkGeneratorCache**.
> 3. A lookup is performed on the internal `ConcurrentSizedTimeoutCache`.
> 4. **Cache Hit:** The existing `CoreDataCacheEntry` is retrieved. If the specific data (e.g., height) is not yet populated, the lazy-evaluation function is called, the entry is updated, and the value is returned to the ChunkGenerator.
> 5. **Cache Miss:** The `computeValue` function is triggered, which calls the expensive `ZoneBiomeResultFunction` provided at construction. A new `CoreDataCacheEntry` is created and stored. The flow then proceeds as a cache hit.

