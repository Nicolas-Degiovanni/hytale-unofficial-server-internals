---
description: Architectural reference for ConcurrentSizedTimeoutCache
---

# ConcurrentSizedTimeoutCache

**Package:** com.hypixel.hytale.server.worldgen.util.cache
**Type:** Transient

## Definition
```java
// Signature
public class ConcurrentSizedTimeoutCache<K, V> implements Cache<K, V> {
```

## Architecture & Concepts
The ConcurrentSizedTimeoutCache is a high-performance, bounded, in-memory cache designed for multi-threaded server environments. Its primary purpose is to store frequently accessed, computationally expensive objects for a limited duration, reducing redundant work in systems like world generation.

The core architectural pattern is **striped locking** (also known as lock striping or sharding). Instead of a single global lock protecting all cache data, the keyspace is partitioned across an array of internal **Buckets**. A key's hash code determines which bucket it belongs to, and each bucket has its own independent lock. This design dramatically reduces lock contention, as threads operating on different keys are highly likely to interact with different buckets, allowing for true parallel access.

Each Bucket instance encapsulates a map, an object pool for cache entries, and a **StampedLock**. The use of StampedLock is a key performance optimization, allowing for non-blocking, optimistic reads. Writes acquire a full exclusive lock, but only for the small segment of the cache contained within that bucket.

To minimize garbage collection overhead, each Bucket maintains a pool of reusable CacheEntry objects. When an entry is evicted, instead of being discarded, it is reset and returned to the pool for future use.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor. The caller must provide critical configuration parameters: capacity, concurrency level, timeout, and behavioral functions for value computation and destruction. It is not managed by a dependency injection framework and is not a singleton.

- **Scope:** The cache lives as long as the object that created it maintains a strong reference. Its operational lifecycle is managed internally by a scheduled task. A `ScheduledFuture` is submitted to the global HytaleServer.SCHEDULED_EXECUTOR upon creation, which periodically invokes the cache's cleanup logic.

- **Destruction:**
    - **Explicit:** The `shutdown()` method must be called for a clean and immediate teardown. This cancels the scheduled cleanup task and synchronously clears all buckets, invoking the provided destroyer function on all remaining entries.
    - **Implicit (GC-driven):** The class uses a `java.lang.ref.Cleaner`. If the cache instance becomes eligible for garbage collection, the Cleaner will automatically cancel the associated `ScheduledFuture`. This is a critical safety net to prevent resource leaks (i.e., orphaned threads in the server's executor service). **WARNING:** Relying on this mechanism is not recommended for deterministic shutdown. Always call `shutdown()` explicitly.

## Internal State & Concurrency
- **State:** The cache's state is highly mutable, consisting of the internal maps within each bucket which are constantly modified by get, cleanup, and shutdown operations.

- **Thread Safety:** This class is fully thread-safe and designed for high-concurrency access.
    - **Sharded Locks:** As described in the architecture, sharding keys across multiple buckets with independent `StampedLock` instances is the primary mechanism for enabling concurrent operations.
    - **Volatile Access:** The timestamp of a CacheEntry, which is updated on every read access, is managed via a `VarHandle`. This ensures that writes to the timestamp have volatile semantics, guaranteeing that updates are immediately visible to all threads, including the background cleanup thread. This prevents race conditions where an entry is incorrectly evicted due to stale timestamp visibility.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(K key) | V | O(1) avg | Retrieves the value for the given key. If the key is not present, it atomically computes, stores, and returns the new value. Throws IllegalStateException if the cache has been shut down. |
| shutdown() | void | O(N) | Immediately cancels the background cleanup task and clears all entries from the cache, invoking the destroyer for each. This is the primary method for releasing cache resources. |
| cleanup() | void | O(N) | Manually triggers a single cleanup cycle, evicting all expired entries. This is normally handled by the background task but can be invoked directly if needed. |

## Integration Patterns

### Standard Usage
The cache should be instantiated with appropriate parameters for the specific use case and held as a field in a long-lived service or manager. The `shutdown` method should be tied to the lifecycle of the owning object.

```java
// Example: Caching computed terrain data
// 1. Define the cache during service initialization
Cache<ChunkPosition, TerrainData> terrainCache = new ConcurrentSizedTimeoutCache<>(
    1024, // capacity
    16,   // concurrency level
    5, TimeUnit.MINUTES, // timeout
    pos -> pos, // The key is the position itself
    this::computeTerrainDataFor, // Function to generate data on miss
    (pos, data) -> data.releaseResources() // Function to clean up data on eviction
);

// 2. Use the cache in a hot path
public TerrainData getTerrain(ChunkPosition pos) {
    return this.terrainCache.get(pos);
}

// 3. Ensure shutdown is called when the service is destroyed
public void onServiceShutdown() {
    this.terrainCache.shutdown();
}
```

### Anti-Patterns (Do NOT do this)
- **Forgetting Shutdown:** Failing to call `shutdown()` on a cache that is no longer needed. While the `Cleaner` provides a safety net, it is non-deterministic and can lead to resource contention in the global executor service.
- **Long-Running Computation:** Providing a `computeValue` function that performs blocking I/O or takes a significant amount of time to complete. This function executes while a write lock is held on a bucket, which will block all other threads attempting to read from or write to that same cache segment.
- **Unstable Keys:** Using a key type K whose `hashCode()` or `equals()` implementation is not stable or correct. This will lead to unpredictable cache behavior, including memory leaks and lost entries.

## Data Pipeline
The data flow for a cache miss is a multi-step, lock-managed process designed for correctness and performance.

> Flow (Cache Miss):
> `get(key)` -> Hash Key -> Select Bucket -> Acquire Read Lock -> **Map Lookup Fails** -> Release Read Lock -> Acquire Write Lock -> Execute `computeValue` function -> Create/Recycle CacheEntry -> **Put Entry in Map** -> Release Write Lock -> Return New Value

