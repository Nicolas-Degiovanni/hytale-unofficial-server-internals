---
description: Architectural reference for WrappedBiCoordinateCache
---

# WrappedBiCoordinateCache<T>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.bicoordinatecache
**Type:** Transient

## Definition
```java
// Signature
public class WrappedBiCoordinateCache<T> implements BiCoordinateCache<T> {
```

## Architecture & Concepts
The WrappedBiCoordinateCache is a specialized, fixed-size 2D data structure designed for high-performance caching in systems that operate on toroidal, or "wrapping", coordinate spaces. Its primary application is within the world generation pipeline, where it serves as a memoization layer for computationally expensive values like biome data, height maps, or noise samples.

The key architectural feature is its coordinate wrapping. Any given X, Z coordinate is mapped via a modulo operation into a local, bounded 2D array index. This allows systems to query for data in a seamless, infinitely repeating world without needing to manage complex boundary conditions or load adjacent regions. For example, when generating a block at coordinate (0, 0), the generator can query for data at (-1, -1) and the cache will correctly map this to an internal coordinate within its fixed-size grid.

This component is fundamentally a performance optimization tool. It trades memory (to store the cached values) for CPU cycles (by avoiding re-computation). It uses a parallel boolean array, *populated*, to track which cells contain valid data, allowing the cache to correctly store null values.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor: `new WrappedBiCoordinateCache(sizeX, sizeZ)`. It is not a managed service and is not retrieved from a central registry. Ownership belongs entirely to the system that instantiates it, typically a short-lived generator or processor.
- **Scope:** The lifetime of a WrappedBiCoordinateCache is strictly bound to its owner. For example, a world generation task might create a cache for a specific region, use it to generate all necessary data, and then discard it upon completion.
- **Destruction:** The object is eligible for garbage collection as soon as its owner releases all references. The `flush()` method provides a mechanism to clear and reuse the cache for a new task without the overhead of reallocation, but the lifecycle remains under the owner's explicit control.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. It consists of two 2D arrays, `values` and `populated`, and an integer `size` counter. The `populated` array acts as a presence map, which is critical for distinguishing between a cached *null* value and an unpopulated cache entry.
- **Thread Safety:** **This class is not thread-safe.** All methods that modify internal state (`save`, `flush`) do so without any synchronization. Concurrent writes from multiple threads will lead to race conditions, data corruption, and an inconsistent `size` count. Read operations (`get`, `isCached`) during a concurrent write may observe a partially updated state.

**WARNING:** Any use of this class in a multi-threaded context requires external synchronization. The calling system must implement its own locking mechanism to guarantee that only one thread can access a given cache instance at a time.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int x, int z) | T | O(1) | Retrieves the cached value for the given coordinates. Throws IllegalStateException if no value is cached at that location. |
| isCached(int x, int z) | boolean | O(1) | Checks if a value has been cached for the given coordinates. This should always be checked before calling get. |
| save(int x, int z, T value) | T | O(1) | Caches the given value at the specified coordinates, overwriting any previous entry. |
| flush(int x, int z) | void | O(1) | Invalidates and removes a single entry from the cache. |
| flush() | void | O(N*M) | Invalidates the entire cache, clearing all entries. This is used to reset the cache for reuse. |
| size() | int | O(1) | Returns the total number of items currently stored in the cache. |

## Integration Patterns

### Standard Usage
The intended use is for a single-threaded process to populate and then query the cache to avoid redundant work. The caller is responsible for handling the case where data is not yet in the cache.

```java
// Example: Caching expensive noise values during world generation
int cacheSize = 16;
WrappedBiCoordinateCache<Float> noiseCache = new WrappedBiCoordinateCache<>(cacheSize, cacheSize);

// World coordinates
int worldX = 123;
int worldZ = 456;

float noiseValue;
if (noiseCache.isCached(worldX, worldZ)) {
    noiseValue = noiseCache.get(worldX, worldZ);
} else {
    noiseValue = NoiseGenerator.calculateExpensiveNoise(worldX, worldZ);
    noiseCache.save(worldX, worldZ, noiseValue);
}
```

### Anti-Patterns (Do NOT do this)
- **Unchecked Access:** Never call `get(x, z)` without first confirming the entry exists with `isCached(x, z)`. Relying on a `try-catch` block to handle a cache miss is inefficient and violates the class's contract.
- **Concurrent Modification:** Do not share a single instance of this cache across multiple threads without an external locking mechanism. This will inevitably lead to data corruption.

```java
// BAD: Potential for IllegalStateException
// This code assumes the value will always be present.
float noise = noiseCache.get(worldX, worldZ);

// BAD: Race condition
// Two threads trying to save to the cache at the same time.
// The final state of values[x][z], populated[x][z], and size is unpredictable.
new Thread(() -> cache.save(1, 1, valueA)).start();
new Thread(() -> cache.save(1, 1, valueB)).start();
```

## Data Pipeline
This class does not process data in a pipeline; it acts as a terminal data store or a memoization side-channel. The flow is initiated and controlled by an external system.

> **Flow 1: Cache Miss**
> World Generator -> `isCached()` returns false -> Calculate Value -> `save()` -> **WrappedBiCoordinateCache**

> **Flow 2: Cache Hit**
> World Generator -> `isCached()` returns true -> `get()` -> **WrappedBiCoordinateCache** -> Use Value

