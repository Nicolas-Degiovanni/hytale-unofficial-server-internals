---
description: Architectural reference for FileMaskCache
---

# FileMaskCache

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Transient

## Definition
```java
// Signature
public class FileMaskCache<T> {
```

## Architecture & Concepts
The FileMaskCache is a specialized, in-memory caching utility designed to optimize performance during server-side world generation. Its primary function is to reduce redundant disk I/O and JSON parsing operations when loading world generation assets, often referred to as "file masks".

This component maintains two distinct internal caches:
1.  A cache for raw **JsonElement** objects, loaded directly from files. This prevents re-reading and re-parsing the same JSON file multiple times within a single generation operation.
2.  A cache for a generic, higher-level type **T**. This is intended to store the fully processed or deserialized representation of the JSON data, preventing redundant processing logic.

It operates as a performance-critical, short-lived data store, tightly coupled to a specific, stateful process like a world generation task. It is not a global or persistent cache.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new FileMaskCache<>()`) by a higher-level manager or loader within the world generation pipeline. It is not managed by a dependency injection framework.
- **Scope:** The lifecycle of a FileMaskCache instance is strictly bound to its owner. It is intended to exist only for the duration of a single, cohesive operation, such as the generation of a specific world region.
- **Destruction:** The object is eligible for garbage collection as soon as its owning loader or process completes and all references to it are dropped. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The FileMaskCache is inherently **mutable**. Its sole purpose is to accumulate and serve data, modifying its internal HashMap state with every `put` or `computeIfAbsent` operation.

- **Thread Safety:** **This class is not thread-safe.** The internal state is managed by standard HashMap collections, which provide no guarantees for concurrent access. If a single instance is shared and modified by multiple threads without external synchronization, it will result in race conditions, lost updates, and potential corruption of the cache.

    **WARNING:** Any system that uses FileMaskCache in a multi-threaded context (e.g., parallel chunk generation) must implement its own locking mechanism around all interactions with the cache instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIfPresentFileMask(String) | T | O(1) avg | Retrieves a processed object from the generic cache. Returns null if not present. |
| putFileMask(String, T) | void | O(1) avg | Inserts a processed object into the generic cache, overwriting any existing entry. |
| cachedFile(String, Function) | JsonElement | O(1) hit, O(N) miss | Atomically gets or computes a JsonElement. On a cache miss, the provided function is executed to load the data, the result is stored, and then returned. |

## Integration Patterns

### Standard Usage
The FileMaskCache should be instantiated at the beginning of a complex, file-intensive operation and passed down to the components that need it.

```java
// A hypothetical loader class that owns the cache
class WorldgenRegionLoader {
    public void loadRegion() {
        // Create a new cache for this specific operation
        FileMaskCache<ProcessedAsset> cache = new FileMaskCache<>();

        // Use the cache to load a raw JSON file
        JsonElement rawData = cache.cachedFile("assets/world/zone1.json", fileName -> loadJsonFromFile(fileName));

        // Process the data and store the result in the other cache
        ProcessedAsset asset = processJson(rawData);
        cache.putFileMask("assets/world/zone1.json", asset);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Singleton Scope:** Do not treat this class as a singleton or store it in a static field for global access. This will create a massive memory leak, as the cache will never be cleared.
- **Concurrent Modification:** Do not share a single instance across multiple threads without an external lock. This will lead to unpredictable behavior and data corruption.
- **Caching Unrelated Data:** While generic, the class is intended for a cohesive set of data related to a single task. Using one instance to cache disparate data from unrelated systems can complicate lifecycle management.

## Data Pipeline
The FileMaskCache acts as an intermediary to prevent expensive operations. The data flow from the perspective of a calling system is as follows.

> Flow (Cache Miss):
> Asset Request -> **FileMaskCache** (Miss) -> Filesystem I/O -> JSON Parser -> **FileMaskCache** (Store) -> Caller

> Flow (Cache Hit):
> Asset Request -> **FileMaskCache** (Hit) -> Caller

