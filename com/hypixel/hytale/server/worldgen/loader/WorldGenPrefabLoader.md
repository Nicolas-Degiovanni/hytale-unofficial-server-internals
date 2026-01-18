---
description: Architectural reference for WorldGenPrefabLoader
---

# WorldGenPrefabLoader

**Package:** com.hypixel.hytale.server.worldgen.loader
**Type:** Transient Service

## Definition
```java
// Signature
public class WorldGenPrefabLoader {
```

## Architecture & Concepts
The WorldGenPrefabLoader serves as a high-performance, caching facade for accessing world generation prefabs from the file system. Its primary architectural role is to abstract the high latency of disk I/O from the core world generation algorithms, which may request the same prefab thousands of times in rapid succession.

It achieves this by wrapping a low-level, synchronous file resolver (PrefabLoader) with a time-based, in-memory cache (TimeoutCache). When a prefab is requested by name, the loader first consults its cache. If a valid entry exists, it is returned immediately. On a cache miss, the loader performs the expensive file system scan, populates the cache with the result, and sets a 30-second expiration on the new entry.

This design pattern ensures that subsequent requests for the same prefab within the 30-second window are served directly from memory, dramatically improving the performance of structure and feature placement during world generation. The class operates on a specific PrefabStoreRoot, making it context-aware for different sets of prefabs (e.g., overworld vs. dungeon-specific).

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generation manager during the initialization of a world or dimension. The constructor requires a PrefabStoreRoot context and the server's root data folder path, indicating it is created by a system with knowledge of the server's file layout.
-   **Scope:** The object's lifetime is typically bound to a single world generation session. It is stateful and intended to be long-lived relative to the generation process to maximize the utility of its internal cache.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and has no explicit `close` or `destroy` method. It is eligible for garbage collection once the owning world generation context is discarded.

## Internal State & Concurrency
-   **State:** This class is stateful. Its primary state is maintained within the internal TimeoutCache instance. The cache is mutable, with entries being added on-demand and evicted automatically after a 30-second timeout.
-   **Thread Safety:** The public API is thread-safe for read operations. The `get` method delegates to the underlying TimeoutCache, which is designed for concurrent access. Multiple world generation worker threads can safely call `get` on a shared WorldGenPrefabLoader instance. The constructor is not thread-safe and the object should be fully constructed in a single thread before being shared.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(String prefabName) | WorldGenPrefabSupplier[] | O(1) to O(N) | Retrieves suppliers for a given prefab. Complexity is O(1) on a cache hit. On a miss, complexity is O(N) where N is the number of files scanned on disk. |
| getStore() | PrefabStoreRoot | O(1) | Returns the prefab store context this loader is bound to. |
| getRootFolder() | Path | O(1) | Returns the root file system path where this loader searches for prefabs. |

## Integration Patterns

### Standard Usage
The intended use is to acquire a single instance of WorldGenPrefabLoader for a given world generation context and share it among all systems that need to place prefabs.

```java
// A world generator obtains the loader from its context
WorldGenPrefabLoader prefabLoader = worldContext.getPrefabLoader();

// A biome placer requests all variants of the "LargeOak" prefab
WorldGenPrefabSupplier[] oakSuppliers = prefabLoader.get("LargeOak");

// The suppliers can now be used to instantiate the prefab
if (oakSuppliers != null && oakSuppliers.length > 0) {
    placePrefab(world, position, oakSuppliers[0]);
}
```

### Anti-Patterns (Do NOT do this)
-   **Repetitive Instantiation:** Do not create a new WorldGenPrefabLoader for each request. This completely defeats the caching mechanism and will lead to severe performance degradation due to constant, redundant disk I/O.

    ```java
    // BAD: Creates a new loader and cache for a single lookup
    WorldGenPrefabLoader loader = new WorldGenPrefabLoader(store, dataPath);
    WorldGenPrefabSupplier[] suppliers = loader.get("somePrefab"); // Cache is immediately discarded
    ```

-   **Ignoring Null Return:** The `get` method can return null if no prefabs match the given name. Code must be robust and handle the null case to prevent NullPointerExceptions.

-   **Assumption of Freshness:** Do not assume the data is always up-to-date with the file system. Due to the 30-second cache, modifications to prefab files on disk will not be reflected until the corresponding cache entry expires. This loader is not suitable for systems requiring real-time prefab reloading.

## Data Pipeline
The flow of data for a single `get` call demonstrates the caching strategy.

> Flow:
> String prefabName -> **WorldGenPrefabLoader.get()** -> TimeoutCache Lookup -> (Cache Miss) -> PrefabLoader.resolvePrefabs() -> File System Scan -> Create WorldGenPrefabSupplier[] -> Populate Cache -> Return to Caller

