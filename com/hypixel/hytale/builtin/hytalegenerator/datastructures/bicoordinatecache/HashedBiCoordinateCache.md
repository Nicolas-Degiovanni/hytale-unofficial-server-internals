---
description: Architectural reference for HashedBiCoordinateCache
---

# HashedBiCoordinateCache

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.bicoordinatecache
**Type:** Transient

## Definition
```java
// Signature
public class HashedBiCoordinateCache<T> implements BiCoordinateCache<T> {
```

## Architecture & Concepts
The HashedBiCoordinateCache is a high-performance, thread-safe data structure designed for caching generic objects, T, against a two-dimensional integer coordinate system (x, z). It serves as a fundamental component within the world generation pipeline, providing a fast lookup mechanism for spatially-indexed data.

Its core design revolves around two key principles:
1.  **Coordinate Hashing:** It transforms a pair of 32-bit integer coordinates (x, z) into a single 64-bit long key. This is achieved by bit-shifting the x-coordinate into the upper 32 bits and adding the z-coordinate to the lower 32 bits. This hashing strategy is deterministic, collision-free, and extremely fast, avoiding the overhead of more complex hashing algorithms or composite key objects.
2.  **Concurrent Storage:** The internal storage is a ConcurrentHashMap, which guarantees thread-safe access without requiring explicit, user-managed locks. This is critical for parallelized systems like the world generator, where multiple worker threads may attempt to read from or write to the cache simultaneously.

This class acts as an in-memory data layer, decoupling expensive data generation logic from data consumption logic. For example, a terrain generation thread can compute and store heightmap data, which can then be safely and efficiently accessed by a separate biome placement or structure spawning thread.

### Lifecycle & Ownership
-   **Creation:** An instance is created directly via its constructor (`new HashedBiCoordinateCache<>()`). It is not managed by a dependency injection framework or a central service registry. Ownership is held by the system that requires spatial caching, such as a WorldGenerator or a RegionProcessor.
-   **Scope:** The lifecycle of a HashedBiCoordinateCache instance is explicitly tied to its owner. It is typically short-lived, existing only for the duration of a specific, large-scale task (e.g., generating a 512x512 world region). It is not a global or session-wide cache.
-   **Destruction:** The object is eligible for garbage collection as soon as its owner is dereferenced. The `flush` methods provide mechanisms for clearing the cache's contents manually, which is essential for memory management during long-running processes.

## Internal State & Concurrency
-   **State:** The class is highly mutable. Its primary state is the internal ConcurrentHashMap, which stores the cached key-value pairs. The size and content of this map change dynamically through `save` and `flush` operations.
-   **Thread Safety:** This class is thread-safe for individual operations. The use of ConcurrentHashMap ensures that methods like `get`, `save`, `isCached`, and `flush(x, z)` can be called from multiple threads without data corruption or the need for external synchronization.

    **WARNING:** Compound operations are not atomic. A sequence such as checking `isCached` and then calling `get` is not a safe atomic unit. A different thread could remove the entry between the two calls, causing the subsequent `get` to fail. The `get` method enforces a fail-fast policy by throwing an IllegalStateException in this scenario.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int x, int z) | T | O(1) avg | Retrieves the value for the given coordinates. Throws IllegalStateException if the key does not exist. |
| isCached(int x, int z) | boolean | O(1) avg | Checks for the existence of a value at the given coordinates without retrieving it. |
| save(int x, int z, T value) | T | O(1) avg | Caches the given value at the specified coordinates. Overwrites any existing value. |
| flush(int x, int z) | void | O(1) avg | Removes the cached entry for the specified coordinates, if it exists. |
| flush() | void | O(N) | Removes all entries from the cache. N is the number of cached items. |
| size() | int | O(N) or O(1) | Returns the number of items in the cache. Complexity depends on the JDK version's ConcurrentHashMap implementation. |

## Integration Patterns

### Standard Usage
This cache is intended to be owned and managed by a higher-level system responsible for a specific task, such as world generation for a particular region.

```java
// A generator creates a cache for its operational scope
BiCoordinateCache<ChunkData> chunkCache = new HashedBiCoordinateCache<>();

// A worker thread generates and saves data
// This operation is thread-safe
chunkCache.save(10, -5, new ChunkData(...));

// Another thread can safely retrieve the data
// Note: This assumes the data is known to exist.
ChunkData data = chunkCache.get(10, -5);
```

### Anti-Patterns (Do NOT do this)
-   **Unbounded Growth:** Creating a single, long-lived HashedBiCoordinateCache for an entire server session is a severe anti-pattern. Without strategic calls to `flush`, this will lead to unbounded memory consumption and eventual server failure as more of the world is explored.
-   **Check-Then-Act:** Relying on `isCached` to guard a call to `get` creates a race condition in a multi-threaded environment.

    ```java
    // ANTI-PATTERN: Race condition
    if (cache.isCached(x, z)) {
        // Another thread could call flush(x, z) here!
        T value = cache.get(x, z); // This may now throw IllegalStateException
        // ...
    }
    ```
    Instead, structure code to handle the potential exception or ensure that data removal only happens at well-defined synchronization points.

## Data Pipeline
The HashedBiCoordinateCache does not process data; it is a terminal state container within a larger pipeline. It acts as a synchronization point and buffer between data producers and consumers.

> **Producer Flow:**
> World Generation Algorithm -> Compute Expensive Data (e.g., Biome) -> **HashedBiCoordinateCache.save(x, z, data)**

> **Consumer Flow:**
> Chunk Population System -> Request Data for (x, z) -> **HashedBiCoordinateCache.get(x, z)** -> Use Data for Spawning Entities

