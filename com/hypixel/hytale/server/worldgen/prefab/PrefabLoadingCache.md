---
description: Architectural reference for PrefabLoadingCache
---

# PrefabLoadingCache

**Package:** com.hypixel.hytale.server.worldgen.prefab
**Type:** Service Component

## Definition
```java
// Signature
public class PrefabLoadingCache {
```

## Architecture & Concepts
The PrefabLoadingCache is a critical performance component within the server's world generation pipeline. Its primary function is to act as an in-memory, on-demand cache for prefab data structures, which are read from disk. By caching these structures, it drastically reduces redundant and expensive disk I/O operations that would otherwise occur when the same prefab is needed multiple times during world generation.

Architecturally, this class sits between the high-level world generation logic (which requests prefabs by name or identifier) and the low-level file system access layer. It implements a "get-or-create" pattern using a concurrent map, ensuring that a given prefab is loaded from disk only once per server session, regardless of how many concurrent generation tasks request it.

A key concept is the separation between the cached `PrefabBuffer` and the returned `IPrefabBuffer`. The `PrefabBuffer` represents the single, shared, in-memory representation of the prefab data. The `IPrefabBuffer` is a lightweight, non-owning accessor or view into that data. This design allows numerous concurrent threads to safely read from the same master `PrefabBuffer` without data corruption or the need for expensive locking on read operations.

## Lifecycle & Ownership
- **Creation:** An instance of PrefabLoadingCache is not a global singleton. It is typically instantiated by a higher-level manager, such as a `WorldContext` or `WorldGenerator`, at the beginning of a world generation session.
- **Scope:** The cache's lifetime is tightly coupled to the world or dimension it serves. It persists as long as its parent world is active on the server. Different worlds will have distinct, isolated PrefabLoadingCache instances.
- **Destruction:** The cache is marked for garbage collection when the world it belongs to is unloaded. The owning context is responsible for explicitly calling the `clear` method during the world unload sequence. This is a critical step to release the memory held by all cached `PrefabBuffer` objects. Failure to do so will result in a severe memory leak.

## Internal State & Concurrency
- **State:** The class is highly stateful. Its core state is the `cache` field, a `ConcurrentHashMap` that stores loaded `PrefabBuffer` objects. The memory footprint of this class can grow significantly depending on the number and size of unique prefabs used during world generation.
- **Thread Safety:** This class is designed to be fully thread-safe and is a cornerstone of parallelized world generation.
    - The use of `ConcurrentHashMap` provides a safe foundation for concurrent reads and writes.
    - The `computeIfAbsent` method is the primary mechanism for achieving thread safety. It guarantees an atomic "check, then load" operation, preventing the classic "cache stampede" problem where multiple threads might attempt to load the same resource simultaneously.
    - **Warning:** While the cache itself is thread-safe, the loader function performs blocking disk I/O. Any thread that requests a non-cached prefab will block until the loading operation is complete. This is an expected behavior but must be considered for performance tuning.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPrefabAccessor(prefabSupplier) | IPrefabBuffer | O(1) or O(IO) | Retrieves a read-only accessor for the specified prefab. If the prefab is not cached, this call blocks on disk I/O to load it. |
| clear() | void | O(N) | Releases all cached `PrefabBuffer` objects and empties the cache. Must be called on world unload to prevent memory leaks. |

## Integration Patterns

### Standard Usage
The PrefabLoadingCache should be retrieved from the appropriate context (e.g., the current world's service registry). It is then used by procedural generation systems to efficiently access prefab data.

```java
// Correct usage within a world generation task
WorldContext worldCtx = ...;
PrefabLoadingCache prefabCache = worldCtx.getService(PrefabLoadingCache.class);

// Request an accessor. This is fast if cached, blocks on I/O if not.
WorldGenPrefabSupplier myPrefab = new WorldGenPrefabSupplier("structures/village_house_1");
IPrefabBuffer prefabData = prefabCache.getPrefabAccessor(myPrefab);

// Use the accessor to place the prefab in the world
placePrefab(world, position, prefabData);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabLoadingCache()`. This would create an isolated, empty cache, defeating the purpose of sharing loaded prefabs across the world's generation tasks. Always retrieve the shared instance from the governing context.
- **Bypassing the Cache:** Avoid calling `PrefabBufferUtil.loadBuffer` directly. This circumvents the entire caching mechanism, leading to severe performance degradation due to repeated disk access for the same resource.
- **Long-Lived Accessors:** Do not store the returned `IPrefabBuffer` accessor in long-term data structures. It is a lightweight object intended for immediate use. Request it from the cache when you need it.

## Data Pipeline
The flow of data is initiated by a world generation request and results in prefab block data being made available to the generator.

> Flow:
> World Generator Request -> `WorldGenPrefabSupplier` (Key) -> **PrefabLoadingCache** -> (Cache Miss) -> Filesystem I/O -> `PrefabBuffer` (Cached Value) -> `IPrefabBuffer` (Accessor) -> World Generator


