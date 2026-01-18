---
description: Architectural reference for CaveGeneratorCache
---

# CaveGeneratorCache

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Transient Component

## Definition
```java
// Signature
public class CaveGeneratorCache extends ExtendedCoordinateCache<CaveType, Cave> {
```

## Architecture & Concepts
The CaveGeneratorCache is a specialized, in-memory, read-through cache designed to memoize the results of expensive procedural cave generation. It operates as a critical performance optimization layer within the server's world generation pipeline.

Its primary responsibility is to store generated Cave objects, keyed by both their spatial coordinates (x, y, z) and their specific CaveType. This prevents the world generator from re-computing the complex noise and placement algorithms for the same cave segment multiple times, which is common when adjacent chunks are generated.

The class extends the generic ExtendedCoordinateCache, inheriting the core logic for capacity limiting, time-based expiration, and concurrent access. Its specialization lies in its integration with the ChunkGenerator's thread-local context, which provides a reusable key object. This design choice is a deliberate optimization to reduce object allocation and garbage collection pressure during the highly intensive and parallelized task of world generation.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generation orchestrator, typically during the initialization of a world or dimension. The creator must provide a concrete implementation of the CaveFunction interface, which encapsulates the actual cave generation logic.
- **Scope:** The cache's lifetime is bound to a specific world generation session. It is not a global or static cache. A separate instance would exist for each active world or dimension being generated.
- **Destruction:** The object is eligible for garbage collection when the parent world generator is destroyed. This typically occurs on server shutdown or when a world is fully unloaded from memory. There is no explicit cleanup or shutdown method.

## Internal State & Concurrency
- **State:** The CaveGeneratorCache is highly mutable. Its internal state consists of a collection of computed Cave objects, managed by the underlying caching implementation inherited from its parent class.
- **Thread Safety:** The cache is designed for concurrent access from multiple world generation worker threads. The underlying data store is assumed to be thread-safe. However, a critical aspect of its concurrency model is the use of a thread-local key object retrieved via the localKey method.

    **WARNING:** The localKey method retrieves a key from a thread-local resource pool managed by ChunkGenerator. This implies that any thread accessing this cache **must** have the ChunkGenerator resource context properly initialized for that thread. Failure to do so will result in a NullPointerException and a world generation fault. This pattern minimizes lock contention and memory churn by reusing key objects on a per-thread basis.

## API Surface
The public contract is primarily defined by its constructor and methods inherited from ExtendedCoordinateCache.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CaveGeneratorCache(caveFunction, maxSize, expireAfterSeconds) | constructor | O(1) | Creates a new cache instance. Requires the generation logic, a maximum size, and an expiration time in seconds. |
| get(CaveType, int, int, int) | Cave | O(1) avg | *Inherited.* Retrieves a Cave. If not present, it invokes the provided CaveFunction to compute, store, and return the result. |

## Integration Patterns

### Standard Usage
The cache should be instantiated once per world generator and used throughout the chunk generation process to request cave data. The `compute` method of the CaveFunction will only be called on a cache miss.

```java
// In a WorldGenerator or similar class
CaveGenerator caveGenerator = new CaveGenerator(worldSeed);
CaveGeneratorCache caveCache = new CaveGeneratorCache(
    caveGenerator::generateCaveForCoords, // Method reference to the generation logic
    10000, // Max number of caves to cache
    300    // Expire entries after 5 minutes
);

// Later, in a worker thread generating a chunk
// The context for ChunkGenerator.getResource() must be set
Cave cave = caveCache.get(CaveType.FUNGAL, chunkX, y, chunkZ);
if (cave != null) {
    // Place cave blocks into the chunk
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation in Workers:** Do not create a new CaveGeneratorCache inside a chunk generation worker thread. This defeats the purpose of caching, as the cache would be discarded immediately.
- **Ignoring Thread-Local Context:** Do not attempt to access the cache from a thread that has not been properly initialized with a ChunkGenerator resource context. This will fail.
- **Global Singleton:** Do not treat this cache as a static global singleton. Doing so would cause caves from different worlds (with different seeds) to be mixed, resulting in catastrophic world generation errors.

## Data Pipeline
The component acts as an intermediary between the chunk generator's structural requests and the expensive cave computation logic.

> Flow:
> ChunkGenerator Request -> **CaveGeneratorCache::get** -> Cache Miss -> CaveFunction::compute -> Procedural Noise Calculation -> New Cave Object -> **CaveGeneratorCache** stores result -> Return Cave Object to ChunkGenerator

