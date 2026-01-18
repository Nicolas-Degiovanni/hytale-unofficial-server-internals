---
description: Architectural reference for WorldGenPrefabSupplier
---

# WorldGenPrefabSupplier

**Package:** com.hypixel.hytale.server.worldgen.loader
**Type:** Transient

## Definition
```java
// Signature
public class WorldGenPrefabSupplier implements PrefabSupplier {
```

## Architecture & Concepts

The WorldGenPrefabSupplier is a lightweight handle, or descriptor, that represents a single prefab definition on the filesystem. It is **not** the prefab data itself. Instead, it acts as a crucial intermediary between the asset discovery system and the in-memory world generation cache.

Its primary role is to serve as a stable key for retrieving fully parsed prefab data (an IPrefabBuffer) from a central resource cache. By using the supplier object as a key, the system decouples the act of finding a prefab file from the act of loading and using it.

A secondary, but critical, function is to calculate the complete spatial footprint (the ChunkBounds) of a prefab. This calculation is recursive, traversing the entire hierarchy of nested child prefabs to determine the final bounding box required for placement in the world. This prevents world generation algorithms from needing to understand the complex internal structure of prefabs.

This class is a foundational component of the server's procedural generation pipeline, enabling the dynamic discovery and composition of structures.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the WorldGenPrefabLoader during the server's initial asset discovery phase. For each valid prefab JSON file found on disk, a corresponding WorldGenPrefabSupplier is instantiated.
- **Scope:** An instance persists for the entire server session. It is owned and held by the WorldGenPrefabLoader that created it, typically within an internal collection mapping prefab paths to suppliers.
- **Destruction:** Instances are eligible for garbage collection only when the server is shutting down and the WorldGenPrefabLoader itself is destroyed.

## Internal State & Concurrency
- **State:** The object's state is mutable, but only for the purpose of lazy initialization and caching. The core identifiers—the loader and file path—are immutable after construction. The `prefabName` and `bounds` fields are computed on first request and the results are cached for all subsequent calls.

- **Thread Safety:** **This class is not thread-safe.** The lazy initialization of the `bounds` and `prefabName` fields is not protected by locks or atomic operations.

    **WARNING:** If multiple threads access `getBounds` or `getPrefabName` on the same uninitialized instance concurrently, a race condition will occur. This can lead to redundant, expensive computations and potentially unpredictable state. All access to a shared supplier instance must be externally synchronized, or confined to a single thread, which is the typical pattern within the world generation system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | IPrefabBuffer | O(1) | Retrieves the fully loaded prefab data from a global cache. Returns null if the prefab is not loaded. |
| getBounds(buffer) | IChunkBounds | O(N*M) | Recursively calculates the total chunk bounds of the prefab and all its children. N is the number of child prefabs, M is the nesting depth. The result is cached; subsequent calls are O(1). |
| getPrefabName() | String | O(1) | Returns a resolved, relative path string for the prefab. The result is cached after the first call. |
| getName() | String | O(1) | Returns the raw, platform-dependent path string. |
| getPath() | Path | O(1) | Returns the NIO Path object for the prefab file. |
| getLoader() | WorldGenPrefabLoader | O(1) | Returns a reference to the loader that created this supplier. |

## Integration Patterns

### Standard Usage

The intended pattern is to retrieve a supplier from the WorldGenPrefabLoader, use it to get the cached prefab data, and then use it again to calculate the complete spatial bounds for placement logic.

```java
// Obtain the loader from the world generation context
WorldGenPrefabLoader loader = chunkGenerator.getResource().getPrefabLoader();

// Request suppliers for a specific prefab path
WorldGenPrefabSupplier[] suppliers = loader.get("my_dungeon/entry_hall");

if (suppliers.length > 0) {
    WorldGenPrefabSupplier supplier = suppliers[0];

    // Use the supplier as a key to get the actual prefab data
    IPrefabBuffer prefabData = supplier.get();

    if (prefabData != null) {
        // Calculate the prefab's total footprint, including all children
        IChunkBounds bounds = supplier.getBounds(prefabData);
        
        // ...use bounds for placement validation and logic...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WorldGenPrefabSupplier()`. Doing so detaches it from the asset management and caching systems, rendering it useless. Always retrieve suppliers from the WorldGenPrefabLoader.
- **Unsynchronized Multi-threaded Access:** Calling `getBounds` on a newly created supplier from multiple threads without external locking is a severe race condition. The world generation pipeline typically processes chunks on a single thread, which naturally avoids this issue.

## Data Pipeline

This class acts as a reference point in the data flow from disk to world rendering. It does not transform data itself but enables other systems to locate and process it.

> Flow:
> Filesystem Scan -> **WorldGenPrefabSupplier (Instance Created)** -> PrefabLoader (Loads data into cache using supplier as key) -> WorldGen Algorithm (Calls `supplier.get()`) -> **WorldGenPrefabSupplier (Calculates bounds)** -> Voxel Placement Logic

