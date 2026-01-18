---
description: Architectural reference for SizedTimeoutCache
---

# SizedTimeoutCache

**Package:** com.hypixel.hytale.server.worldgen.util.cache
**Type:** Managed Component

## Definition
```java
// Signature
public class SizedTimeoutCache<K, V> implements Cache<K, V> {
```

## Architecture & Concepts
The SizedTimeoutCache is a high-performance, general-purpose caching component designed for server-side systems, particularly in contexts like world generation where temporary, expensive-to-compute data must be managed efficiently.

It implements a sophisticated dual-eviction strategy, combining two distinct policies to maintain memory discipline:
1.  **Size-Based (LRU):** The cache is bounded by a maximum size. When this limit is exceeded, the **Least Recently Used** entry is evicted. This is achieved internally using an Object2ObjectLinkedOpenHashMap, which maintains insertion order and provides efficient access to the oldest entry.
2.  **Time-Based (Timeout):** Entries are evicted if they have not been accessed for a specified duration. A background task, scheduled on the global HytaleServer.SCHEDULED_EXECUTOR, periodically scans the cache and removes expired entries.

The component is designed as a **read-through cache**. An optional loading function can be provided during construction. When a `get` operation results in a cache miss, this function is automatically invoked to compute or load the value, which is then inserted into the cache before being returned.

To minimize garbage collection overhead in high-throughput scenarios, the cache maintains an internal object pool for its CacheEntry wrappers.

## Lifecycle & Ownership
The lifecycle of a SizedTimeoutCache instance is explicitly managed by its creator. Proper management is critical to prevent resource leaks.

-   **Creation:** An instance is created via its public constructor. The caller must provide all configuration parameters, including size, timeout, and optional loader and destroyer functions. Upon instantiation, it immediately schedules a recurring cleanup task on a server-wide executor.

-   **Scope:** The cache is active and functional as long as a strong reference to it exists. Its operational scope is tied to the HytaleServer executor service.

-   **Destruction:** Explicit cleanup is mandatory.
    -   **Explicit:** The `shutdown()` method **must** be called when the cache is no longer needed. This cancels the background cleanup task and, if a destroyer function is present, disposes of all remaining entries. Failure to call `shutdown` is a severe resource leak.
    -   **Implicit (Safety Net):** The class uses a `java.lang.ref.Cleaner` as a fallback mechanism. If the SizedTimeoutCache object becomes eligible for garbage collection, the Cleaner will automatically cancel the associated background task. This is a safety measure, not a substitute for explicit `shutdown()` calls.

## Internal State & Concurrency
-   **State:** The SizedTimeoutCache is a stateful and highly mutable component. Its primary state is stored in an `Object2ObjectLinkedOpenHashMap` which holds the key-value pairs and their metadata.

-   **Thread Safety:** This class is thread-safe. All public methods that access or modify the internal map and entry pool are synchronized using a coarse-grained lock on the map object itself. While this ensures correctness, it can become a point of contention in environments with extremely high concurrent access. Callers should be aware of this potential bottleneck.

## API Surface
The public contract is designed for standard cache interactions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SizedTimeoutCache(expire, unit, maxSize, func, destroyer) | constructor | O(1) | Creates and initializes the cache, scheduling the background cleanup task. |
| get(K key) | V | O(1) | Retrieves a value. Triggers the loading function on a cache miss if configured. Throws IllegalStateException if the cache is shut down. |
| put(K key, V value) | void | O(1) | Inserts or updates a value in the cache. Throws IllegalStateException if the cache is shut down. |
| shutdown() | void | O(N) | Permanently disables the cache, cancels its background task, and cleans up all entries. This operation is irreversible. |
| getWithReusedKey(K reusedKey, Function<K, K> keyPool) | V | O(1) | A specialized `get` for performance-critical paths where key objects are expensive to allocate, allowing for key object reuse. |

## Integration Patterns

### Standard Usage
The cache must be explicitly shut down, typically in a `finally` block or through a managed lifecycle system, to guarantee resource release.

```java
// Example: Caching computed terrain data
SizedTimeoutCache<ChunkCoord, TerrainData> terrainCache = new SizedTimeoutCache<>(
    30, TimeUnit.SECONDS, 1024,
    this::generateTerrainForChunk, // Loader function
    TerrainData::releaseNativeBuffers // Destroyer function
);

try {
    // ... application logic using terrainCache.get(coord) ...
} finally {
    // CRITICAL: Ensure shutdown is always called
    terrainCache.shutdown();
}
```

### Anti-Patterns (Do NOT do this)
-   **Forgetting Shutdown:** Failure to call `shutdown()` will leave the background cleanup task running indefinitely or until the object is garbage collected. This leaks scheduler resources and prevents timely destruction of cached values.
-   **Blocking Destroyer:** Providing a `destroyer` function that performs long-running or blocking I/O operations. The destroyer is executed while holding the main cache lock, which will block all other threads from accessing the cache.
-   **Ignoring Exceptions:** Calling `get` or `put` after `shutdown` will throw an `IllegalStateException`. Code should not attempt to use a cache that has been shut down.

## Data Pipeline
The data flow for a typical read-through (cache miss) operation illustrates the internal locking and computation sequence.

> Flow:
> `get(key)` invoked -> Thread acquires lock -> Internal map lookup (miss) -> Thread releases lock -> Loader function `func.apply(key)` is executed -> Thread re-acquires lock -> New entry is created/pooled and put into map -> Thread releases lock -> Value is returned to caller.

