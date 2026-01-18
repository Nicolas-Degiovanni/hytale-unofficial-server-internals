---
description: Architectural reference for the BiCoordinateCache interface, a core contract for 2D spatial data caching.
---

# BiCoordinateCache

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.bicoordinatecache
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface BiCoordinateCache<T> {
```

## Architecture & Concepts
The BiCoordinateCache interface defines a strict contract for two-dimensional, key-value caching systems where the key is a pair of integer coordinates (e.g., x, z). It serves as a fundamental abstraction within the world generation and data management pipelines, decoupling high-level systems from the underlying cache implementation and strategy.

This contract is critical for systems that process spatially-organized data, such as terrain generation, biome mapping, or structure placement. By programming against this interface, a system like a WorldGenerator can request data for a coordinate without needing to know if the data is being fetched from memory, disk, or a remote source, or what the eviction policy (e.g., LRU, LFU) might be. This promotes modularity and allows different cache implementations to be swapped in based on performance requirements or the specific context of the data being stored.

## Lifecycle & Ownership
As an interface, BiCoordinateCache is a stateless contract and has no lifecycle of its own. The lifecycle and ownership semantics apply to its concrete implementations.

- **Creation:** Implementations are instantiated by factory classes or service providers responsible for a specific domain. For example, a WorldSessionManager might create a persistent ChunkDataCache that implements this interface.
- **Scope:** The lifetime of an implementation is tied to its owner and purpose. A cache for procedural noise values might be transient and scoped to a single generation task, while a cache for player-modified region data would persist for the entire server session.
- **Destruction:** Cleanup is the responsibility of the creating entity. Implementations that manage off-heap memory or file handles must be explicitly flushed and closed. Consumers of this interface should not assume responsibility for destruction.

## Internal State & Concurrency
The interface itself is stateless. However, all non-trivial implementations are inherently stateful, as their primary purpose is to store and manage data.

- **State:** Implementations are expected to be mutable. They will internally manage a collection of cached values, tracking keys and potentially metadata for eviction policies.
- **Thread Safety:** **WARNING:** The BiCoordinateCache contract provides **no guarantee** of thread safety. Implementations must document their own concurrency guarantees. Systems operating in a multi-threaded context, such as parallelized world generation, must either use a known thread-safe implementation or externally synchronize access to the cache. Failure to do so will result in data corruption and race conditions.

## API Surface
The public API defines the essential operations for a 2D spatial cache.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int, int) | T | O(1) | Retrieves the value for the given coordinate pair. Returns null if not cached. |
| isCached(int, int) | boolean | O(1) | Checks for the existence of a value at the coordinates without retrieving it. |
| save(int, int, T) | T | O(1) | Stores a value at the given coordinates, potentially overwriting an existing entry. Returns the saved value. |
| flush(int, int) | void | O(1) | Invalidates the cache entry for a specific coordinate pair. |
| flush() | void | O(N) | Clears the entire cache, invalidating all entries. Complexity is proportional to the number of cached items. |
| size() | int | O(1) | Returns the total number of items currently stored in the cache. |

## Integration Patterns

### Standard Usage
Systems should depend on the interface, not a concrete implementation. The cache is typically injected or retrieved from a central service registry. This allows the underlying caching strategy to be configured or changed without altering the consuming logic.

```java
// A world generator retrieves a cache for biome data from a context object
BiCoordinateCache<Biome> biomeCache = worldContext.getBiomeCache();

// Check the cache before performing an expensive calculation
if (!biomeCache.isCached(chunkX, chunkZ)) {
    Biome generatedBiome = calculateBiomeFor(chunkX, chunkZ);
    biomeCache.save(chunkX, chunkZ, generatedBiome);
}

Biome targetBiome = biomeCache.get(chunkX, chunkZ);
// ... proceed with block placement using targetBiome
```

### Anti-Patterns (Do NOT do this)
- **Assuming Implementation Details:** Do not write code that relies on the side effects of a specific implementation, such as assuming an LRU eviction policy. The contract only guarantees storage and retrieval.
- **Casting to Concrete Type:** Avoid casting the interface to a concrete class to access non-standard methods. This violates the abstraction and tightly couples your system to a specific cache strategy.
- **Ignoring Concurrency:** Do not share a non-thread-safe cache implementation across multiple threads without external locking. This is a common source of bugs in parallelized generation systems.

## Data Pipeline
BiCoordinateCache acts as a memoization layer, sitting between a data producer (e.g., a noise function) and a data consumer (e.g., a block placement system). It prevents redundant, computationally expensive operations.

> Flow:
> Data Consumer (e.g., TerrainMesher) -> Request for data at (x, z) -> **BiCoordinateCache** -> [Cache Hit] -> Return cached data
>
> Flow:
> Data Consumer (e.g., TerrainMesher) -> Request for data at (x, z) -> **BiCoordinateCache** -> [Cache Miss] -> Data Producer (e.g., NoiseGenerator) -> Generate data -> **BiCoordinateCache**.save(x, z, data) -> Return generated data

