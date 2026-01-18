---
description: Architectural reference for CacheThreadMemory
---

# CacheThreadMemory

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders.cached
**Type:** Transient

## Definition
```java
// Signature
public class CacheThreadMemory {
```

## Architecture & Concepts

CacheThreadMemory is a specialized, high-performance, in-memory caching component designed for the world generation pipeline. Its primary function is to provide a fixed-size, Least Recently Used (LRU) cache for expensive-to-calculate data, specifically arrays of Vector3d positions.

The class name itself provides critical architectural context:
-   **Cache:** It is a temporary storage mechanism to avoid re-computation.
-   **Thread:** It is explicitly designed for single-threaded access. This is a deliberate performance optimization to avoid the overhead of locks or concurrent data structures, which are unnecessary within the isolated context of a world generation worker thread.
-   **Memory:** The cache resides entirely in RAM and is not persisted to disk.

Architecturally, it serves as a performance-critical buffer for `PositionProvider` implementations. When a generator requests a set of positions for a given world section (identified by a `Long` key, likely a packed coordinate), the provider first queries this cache. A cache hit bypasses costly noise calculations or procedural algorithms, significantly accelerating world generation for areas that are frequently accessed during a single generation task.

The LRU eviction policy is implemented using a combination of a HashMap for O(1) average-time lookups and a LinkedList to maintain the access order. When a new item is added to a full cache, the least recently used item (at the head of the list) is evicted.

## Lifecycle & Ownership

-   **Creation:** An instance is created directly via its constructor, `new CacheThreadMemory(int size)`. It is typically instantiated and owned by a higher-level manager or service that requires caching, such as a `CachedPositionProvider`.
-   **Scope:** The object's lifetime is strictly bound to its owner. It persists only for the duration of the parent component's task. For example, if owned by a world generation worker, it is created when the worker starts and discarded when the worker completes its chunk generation task. It is not a global or session-wide singleton.
-   **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as its owning object is no longer referenced. It does not hold any native resources and requires no explicit cleanup method.

## Internal State & Concurrency

-   **State:** This class is highly **mutable**. Its internal `sections` map and `expirationList` are continuously modified during `get` and `put` operations. The state represents a snapshot of the most recently used position data up to its configured capacity.
-   **Thread Safety:** This class is **not thread-safe**. It uses non-synchronized collections (`HashMap`, `LinkedList`) and contains no locking mechanisms. Concurrent access from multiple threads will result in unpredictable behavior, data corruption, and likely a `ConcurrentModificationException`.

    **WARNING:** This component MUST NOT be shared across threads. Each worker thread requiring a cache must maintain its own private instance of CacheThreadMemory.

## API Surface

The public API is minimal, focused on the core caching operations. The following methods are inferred from its design as a standard LRU cache.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(Long key) | Vector3d[] | O(1) | Retrieves an entry. On a hit, the entry is moved to the end of the expiration list. Returns null on a miss. |
| put(Long key, Vector3d[] value) | void | O(1) | Adds or updates an entry. If the cache is full, the least recently used entry is evicted. |

## Integration Patterns

### Standard Usage

The intended use is for a parent service to create, own, and manage the cache for its own internal operations. The caller is responsible for handling cache misses by computing the required data and populating the cache.

```java
// A generator or provider creates a cache for its lifetime
int CACHE_SIZE = 256;
CacheThreadMemory positionCache = new CacheThreadMemory(CACHE_SIZE);

// On a request for data
long sectionKey = getSectionKeyFor(x, z);
Vector3d[] positions = positionCache.get(sectionKey);

if (positions == null) {
    // Cache miss: compute the data
    positions = expensiveCalculationFor(sectionKey);
    // Populate the cache for next time
    positionCache.put(sectionKey, positions);
}

// Use the positions array
processPositions(positions);
```

### Anti-Patterns (Do NOT do this)

-   **Sharing Across Threads:** Never pass an instance of CacheThreadMemory to another thread or store it in a static field accessible by multiple threads. This will break the non-concurrent contract and lead to data corruption.
-   **Over-Sizing:** Instantiating the cache with an excessively large size can lead to high memory pressure and potential `OutOfMemoryError`, especially with many concurrent world generation threads. The size should be carefully tuned based on performance profiling.
-   **Zero-Size Cache:** Creating a cache with `new CacheThreadMemory(0)` is valid but functionally useless. It will immediately evict any item that is put into it.

## Data Pipeline

CacheThreadMemory acts as a conditional branch within a larger data flow. It either short-circuits the flow by providing a cached result or allows it to proceed to the more expensive computation stage.

> Flow on Cache Miss:
> Position Request (Key) -> **CacheThreadMemory.get()** -> *null* -> Procedural Generator -> Compute Positions -> **CacheThreadMemory.put()** -> Return Positions

> Flow on Cache Hit:
> Position Request (Key) -> **CacheThreadMemory.get()** -> *Vector3d[]* -> Return Positions

