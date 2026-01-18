---
description: Architectural reference for CoreDataCacheEntry
---

# CoreDataCacheEntry

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Transient Data Object

## Definition
```java
// Signature
public class CoreDataCacheEntry implements Function<CoreDataCacheEntry, CoreDataCacheEntry> {
```

## Architecture & Concepts
CoreDataCacheEntry is a fundamental data structure, not a service, acting as a container for the computed results of a single, discrete unit of the world during server-side world generation. It represents the "core data" for a vertical column of the world, holding essential values like biome information, heightmap noise, and final block height.

This class is designed to be the value type in a larger, grid-based caching system, likely named CoreDataCache. The world generation pipeline is computationally expensive; by caching these entry objects, the engine avoids redundant calculations for world columns that have already been processed.

The implementation of the `Function<CoreDataCacheEntry, CoreDataCacheEntry>` interface is a critical design choice. It provides a standardized mechanism, via the `apply` method, to update an existing entry in-place. This pattern is highly indicative of an object pooling strategy, where entries are recycled to reduce garbage collection overhead in the performance-critical world generation loop.

## Lifecycle & Ownership
- **Creation:** Instances are created and managed exclusively by a parent caching system (e.g., CoreDataCache). A new entry is typically instantiated when the cache encounters a miss for a specific world coordinate. In high-performance scenarios, these objects are pre-allocated and managed in an object pool.

- **Scope:** The lifetime of a CoreDataCacheEntry is strictly tied to its parent cache. It persists in memory as long as the data for its corresponding world coordinate is deemed relevant by the cache's eviction policy.

- **Destruction:** An entry is eligible for garbage collection when it is evicted from its parent cache. If an object pooling pattern is in use, the object is not destroyed but rather reset via the `apply` method and returned to the pool for reuse.

## Internal State & Concurrency
- **State:** This object is highly **mutable**. Its primary purpose is to be populated with data as world generation stages complete. The `zoneBiomeResult` field is final, but the object it references is itself mutable. All other data fields are volatile and designed for modification post-instantiation.

- **Thread Safety:** This class is **not thread-safe** for concurrent modification. The `volatile` keyword on its fields ensures that writes from one thread are visible to other threads, which is essential for a multi-threaded world generator. However, `volatile` does not provide atomicity.

    **Warning:** Modifying an entry from multiple threads without external synchronization will lead to data corruption and unpredictable world generation outcomes. All write access must be synchronized by the owning system, typically the parent cache, which should enforce a single-writer policy for any given entry.

## API Surface
The public fields of this class constitute its primary API, indicating its role as a transparent data container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(entry) | CoreDataCacheEntry | O(1) | Mutates this instance by copying all data from the provided entry. Returns a reference to this instance for chaining. |

## Integration Patterns

### Standard Usage
This object should never be managed directly by client code. It is retrieved from a higher-level service or cache, its data is read, and then it is discarded. The generator populates it; consumers read from it.

```java
// Hypothetical world generator accessing a cached entry
CoreDataCache cache = world.getCoreDataCache();
CoreDataCacheEntry columnData = cache.getEntryAt(chunkX, chunkZ);

if (columnData != null) {
    int height = columnData.height;
    Biome biome = columnData.zoneBiomeResult.biome;
    // ... use the pre-computed data
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new CoreDataCacheEntry()` within a generation loop. This defeats the caching and object pooling mechanisms, leading to severe performance degradation and high garbage collection pressure.

- **Concurrent Modification:** Do not allow multiple threads to write to the same CoreDataCacheEntry instance simultaneously. The owning cache must implement a locking strategy to ensure safe publication of generated data.

- **Stale Reads:** Do not read from an entry that may be in a partially computed state. Always ensure the generation stage responsible for populating the required fields has completed before accessing them. Check for sentinel values like `NO_HEIGHT`.

## Data Pipeline
CoreDataCacheEntry is a critical waypoint in the world generation data flow. It acts as a sink for various computational stages and a source for subsequent stages that build upon its data.

> Flow:
> WorldGen Request (x, z) -> CoreDataCache Lookup -> **CoreDataCacheEntry** (Cache Miss) -> Biome & Noise Calculators -> Populate **CoreDataCacheEntry** -> Store in Cache -> Return to Caller

