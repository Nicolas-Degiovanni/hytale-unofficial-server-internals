---
description: Architectural reference for LocalCachedChunkAccessor
---

# LocalCachedChunkAccessor

**Package:** com.hypixel.hytale.server.core.universe.world.accessor
**Type:** Transient

## Definition
```java
// Signature
public class LocalCachedChunkAccessor implements OverridableChunkAccessor<WorldChunk> {
```

## Architecture & Concepts
The LocalCachedChunkAccessor is a high-performance implementation of the **Decorator** design pattern, acting as a specialized, bounded cache for world chunk data. It wraps a primary ChunkAccessor, referred to as the *delegate*, to intercept chunk requests and serve them from a local, in-memory array.

Its primary function is to accelerate world-access operations that exhibit high spatial locality, such as world generation feature placement, localized physics simulations, or complex structure validation. By maintaining a small, hot cache of chunks in a contiguous array, it dramatically reduces the overhead of repeated lookups into the main world chunk map, which may be a more complex and less cache-friendly data structure.

The system operates on a **cache-aside** basis. When a chunk is requested:
1.  The accessor first checks if the coordinates fall within its cached region.
2.  If inside the region, it attempts to retrieve the chunk from its internal array.
3.  On a cache hit, the stored chunk is returned immediately.
4.  On a cache miss, the request is forwarded to the delegate. The delegate's response is then stored in the local cache before being returned to the caller.
5.  If the coordinates are outside the cached region, the request is passed directly to the delegate without any caching behavior.

This design provides a transparent performance layer for localized tasks without modifying the underlying global chunk management system.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively through its static factory methods, such as atChunkCoords or atWorldCoords. It is **not** managed by a dependency injection container and should be instantiated on-demand by the system that requires it. A common creator is a world generation feature pass that needs to operate on a small section of the world.

-   **Scope:** The object is designed to be **short-lived and method-scoped**. It should exist only for the duration of a single, well-defined operation. For example, an instance might be created at the start of a function to generate a tree, used to modify blocks within a 3x3 chunk area, and then be discarded when the function returns.

-   **Destruction:** Destruction is managed by the Java Garbage Collector. There are no explicit cleanup or close methods. Once all references to an instance are dropped, it becomes eligible for garbage collection.

**WARNING:** Holding a long-term reference to a LocalCachedChunkAccessor can lead to memory leaks. Its internal cache will retain references to WorldChunk objects, preventing them from being unloaded by the main world system.

## Internal State & Concurrency
-   **State:** The internal state is highly **mutable**. The primary state is the chunks array, which is lazily populated as chunk requests are processed. Its contents are entirely dependent on the sequence of API calls made against it.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. Accessing a single instance from multiple threads concurrently will result in race conditions, data corruption in the cache, and non-deterministic behavior. It must be confined to the thread that created it.

## API Surface
The public API is designed for creating, populating, and querying the local chunk cache.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| atChunkCoords(delegate, centerX, centerZ, radius) | static LocalCachedChunkAccessor | O(R²) | **Primary Factory.** Creates an accessor for a square region of chunks. Complexity is for array allocation. |
| getChunk(index) | WorldChunk | O(1) | Retrieves a chunk. On cache miss, calls the delegate, which may involve disk I/O. |
| getChunkIfInMemory(x, z) | WorldChunk | O(1) | Retrieves a chunk only if it is already loaded in memory by the delegate. Will not trigger disk I/O. |
| getChunkIfLoaded(x, z) | WorldChunk | O(1) | Retrieves a chunk only if it is loaded and marked as ticking. Returns null otherwise. |
| overwrite(wc) | void | O(1) | **CRITICAL:** Forcefully inserts a chunk into the cache, replacing any existing entry. Bypasses the delegate. |
| cacheChunksInRadius() | void | O(N²) | Eagerly populates the entire cache by querying the delegate for all chunks within its bounds. |

## Integration Patterns

### Standard Usage
The intended use case is for a transient, localized operation. A system creates the accessor, performs its work within the cached region, and then discards the accessor.

```java
// A world generator feature creates an accessor for its area of interest.
ChunkAccessor primaryAccessor = world.getChunkAccessor();
LocalCachedChunkAccessor cache = LocalCachedChunkAccessor.atChunkCoords(primaryAccessor, centerX, centerZ, 3);

// The feature can now perform many cheap lookups.
for (int x = -16; x <= 16; x++) {
    for (int z = -16; z <= 16; z++) {
        WorldChunk chunk = cache.getChunkIfInMemory(chunkX, chunkZ);
        if (chunk != null) {
            // ... perform block modifications
        }
    }
}

// When the method finishes, 'cache' goes out of scope and is garbage collected.
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Caching:** Do not store an instance of LocalCachedChunkAccessor in a field of a long-lived service or entity. It is not designed for persistent caching and will prevent chunks from being unloaded.

-   **Shared State:** Do not pass an instance between different threads or systems. Its non-thread-safe nature makes it unsuitable for concurrent access. Each thread performing a localized task should create its own instance.

-   **Direct Instantiation:** The constructor is protected for a reason. Do not attempt to subclass or use reflection to call `new LocalCachedChunkAccessor()`. Always use the provided static factory methods to ensure correct initialization.

## Data Pipeline
The LocalCachedChunkAccessor acts as an intermediary in the chunk request pipeline. It either terminates the request chain with a cache hit or forwards it to the next accessor in the chain.

> **Flow (Cache Hit):**
>
> World System -> getChunk(x, z) -> **LocalCachedChunkAccessor** -> (Chunk is in local array) -> Return WorldChunk

> **Flow (Cache Miss):**
>
> World System -> getChunk(x, z) -> **LocalCachedChunkAccessor** -> (Chunk not in local array) -> Delegate ChunkAccessor -> (Potentially Disk I/O) -> Return WorldChunk -> **LocalCachedChunkAccessor** (stores chunk in array) -> Return WorldChunk

