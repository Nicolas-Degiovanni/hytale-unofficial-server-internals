---
description: Architectural reference for TimeoutCache
---

# TimeoutCache

**Package:** com.hypixel.hytale.server.worldgen.util.cache
**Type:** Transient Component

## Definition
```java
// Signature
public class TimeoutCache<K, V> implements Cache<K, V> {
```

## Architecture & Concepts
The TimeoutCache is a high-performance, thread-safe, in-memory caching component designed for ephemeral data. Its primary architectural role is to reduce computational or I/O load in performance-critical systems, such as world generation, by memoizing the results of expensive operations.

It implements a **read-through** caching strategy combined with a **time-to-live (TTL)** eviction policy. When a value is requested via the *get* method:
1.  If the value exists and is not expired, its timestamp is updated (**touch-on-read**), extending its lifetime, and the value is returned.
2.  If the value does not exist, a developer-provided factory function is synchronously invoked to generate it. The result is stored and returned.

Eviction is handled by a dedicated background task scheduled on the server's shared `SCHEDULED_EXECUTOR`. This active cleanup mechanism ensures that stale entries are periodically purged, preventing unbounded memory growth.

## Lifecycle & Ownership
The lifecycle of a TimeoutCache instance is critical to system stability and resource management.

-   **Creation:** An instance is created directly via its constructor: `new TimeoutCache(...)`. The creator is responsible for defining the cache's behavior by supplying an expiration duration, a value-generation function (*Function*), and an optional resource-destruction callback (*BiConsumer*).

-   **Scope:** The cache persists as long as a strong reference to it is maintained. Its lifetime is typically bound to a parent component, such as a specific world instance or a player session manager. It is not a global singleton.

-   **Destruction:**
    -   **Explicit (Recommended):** The `shutdown()` method must be called to guarantee deterministic cleanup. This immediately cancels the background cleanup task, iterates through all remaining entries, and invokes the destruction callback on each one. This is the required pattern for components with a well-defined shutdown sequence.
    -   **Implicit (Safety Net):** In the event that `shutdown()` is not called and the cache instance becomes eligible for garbage collection, a `java.lang.ref.Cleaner` is employed. This modern finalization mechanism ensures the scheduled background task is cancelled, preventing a permanent resource leak from the executor service. **WARNING:** Relying on this mechanism is considered an anti-pattern, as garbage collection is non-deterministic and will not clean up the *values* within the cache.

## Internal State & Concurrency
-   **State:** The TimeoutCache is a stateful and highly mutable component. Its core state is a `ConcurrentHashMap` that stores cache entries. Timestamps of these entries are updated on every read access, and the map's structure is modified by both client threads (via `get`) and the background cleanup thread.

-   **Thread Safety:** This class is fully thread-safe and designed for high-concurrency environments.
    -   All map operations are delegated to `ConcurrentHashMap`, which provides lock-free reads and fine-grained locking for writes.
    -   The `get` method uses the atomic `compute` operation to prevent race conditions where multiple threads might attempt to generate the same missing key simultaneously.
    -   The background `cleanup` task uses a conditional `remove` operation to safely evict an entry only if it has not been modified by another thread in the interim.
    -   No external synchronization is required when interacting with a TimeoutCache instance.

## API Surface
The public contract is minimal, focusing on core cache operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(K key) | V | O(1) avg | Retrieves the value for the given key. Atomically generates, caches, and returns the value if not present. Throws IllegalStateException if the cache has been shut down. |
| shutdown() | void | O(N) | Immediately cancels the cleanup task and purges all entries from the cache, invoking the destroyer on each. This is a terminal operation. |
| cleanup() | void | O(N) | Scans the cache and removes all expired entries. This is intended for internal use by the scheduled executor but can be invoked manually to force a purge. |

## Integration Patterns

### Standard Usage
The TimeoutCache should be instantiated and managed by a higher-level service that controls its lifecycle.

```java
// Example: Caching expensive terrain data for a world region
Function<RegionCoords, TerrainData> terrainGenerator = (coords) -> {
    // Expensive operation: generate or load from disk
    return generateTerrainFor(coords);
};

BiConsumer<RegionCoords, TerrainData> terrainDestroyer = (coords, data) -> {
    // Optional: clean up native resources if TerrainData holds any
    data.release();
};

// Create a cache that expires entries after 5 minutes of inactivity
Cache<RegionCoords, TerrainData> regionCache = new TimeoutCache<>(
    5, TimeUnit.MINUTES,
    terrainGenerator,
    terrainDestroyer
);

// In the game loop or a request handler:
TerrainData data = regionCache.get(currentRegionCoords);
// ... use data

// When the world unloads, ensure the cache is shut down
// world.onUnload(() -> regionCache.shutdown());
```

### Anti-Patterns (Do NOT do this)
-   **Leaking the Cache:** Failing to call `shutdown()` when the owning component is destroyed. While the `Cleaner` prevents the background thread from leaking, it does not release the memory held by the cached values. This can lead to significant memory leaks.
-   **Long-Running Generator Functions:** The `Function` provided for value generation is executed synchronously on the thread calling `get`. A long-running function will block the caller, potentially causing server stalls or degraded performance. Offload complex operations to a separate worker pool if necessary.
-   **Ignoring `IllegalStateException`:** Accessing `get` after `shutdown` has been called will throw an exception. Code must not attempt to use a cache that has been terminated.

## Data Pipeline

The component has two primary flows: one for data retrieval and one for data eviction.

> **Retrieval Flow (Cache Miss):**
> Thread calls `get(key)` -> `ConcurrentHashMap.compute` finds no entry -> `Function` is invoked -> New `CacheEntry` is created with current timestamp -> Entry is stored in map -> Value is returned to caller.

> **Eviction Flow:**
> `HytaleServer.SCHEDULED_EXECUTOR` triggers -> `CleanupRunnable` executes -> `cleanup()` is called -> Map is iterated -> Entry timestamp is compared to `(now - timeout)` -> Expired entry is removed via atomic `map.remove` -> `BiConsumer` destroyer is invoked with key and value.

