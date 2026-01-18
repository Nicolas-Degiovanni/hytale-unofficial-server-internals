---
description: Architectural reference for PrefabStore
---

# PrefabStore

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Singleton

## Definition
```java
// Signature
public class PrefabStore {
```

## Architecture & Concepts

The PrefabStore is the centralized, server-side service for managing the lifecycle of game world structures known as "prefabs". It functions as a high-performance, caching facade over the server's file system, abstracting the underlying storage complexity from game logic. Its primary responsibility is to load, deserialize, cache, and save prefab data, which is represented in-memory as BlockSelection objects.

This system is designed to resolve prefabs from multiple, layered sources, providing a clear hierarchy for default assets, world-specific overrides, and server-level configurations:

1.  **Asset Prefabs:** Sourced from the core game assets or loaded from mod AssetPacks. These are typically read-only base structures.
2.  **WorldGen Prefabs:** Stored within a specific world's generation data. These are used by procedural generation systems and can be unique to a world seed or configuration.
3.  **Server Prefabs:** General-purpose prefabs stored at the server's root level, intended for use by server administrators, commands, or plugins.

The core of the PrefabStore is an in-memory, concurrent cache that dramatically reduces disk I/O and deserialization overhead for frequently accessed prefabs. Once a prefab is loaded from a file, it is retained in the cache for subsequent requests, ensuring near-instantaneous access.

### Lifecycle & Ownership

-   **Creation:** The PrefabStore is instantiated eagerly as a static final field, INSTANCE, upon the first time the class is loaded by the JVM. This guarantees that a single, globally accessible instance exists before any game systems attempt to use it.
-   **Scope:** Global and application-scoped. The single instance persists for the entire lifetime of the server process. Its internal cache grows as more prefabs are requested.
-   **Destruction:** The PrefabStore is never explicitly destroyed. The singleton instance and its associated cache are reclaimed by the Java Virtual Machine during the final stages of server shutdown.

## Internal State & Concurrency

-   **State:** The internal state of the PrefabStore is highly mutable. Its primary state is the PREFAB_CACHE, a map that is continuously populated with BlockSelection objects as they are loaded from disk. The state of the cache is a direct reflection of the prefab files that have been accessed during the server's runtime.
-   **Thread Safety:** This class is thread-safe for all public operations. The internal cache is implemented with a ConcurrentHashMap, which guarantees safe concurrent read and write access without explicit locking. The primary loading method, getPrefab, uses the atomic `computeIfAbsent` operation. This prevents race conditions where multiple threads might attempt to load the same prefab from disk simultaneously, ensuring that the file is read and deserialized only once.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | PrefabStore | O(1) | Returns the global singleton instance. |
| getPrefab(path) | BlockSelection | O(N) | Loads a prefab from an absolute path. O(1) on cache hit; O(N) on cache miss, where N is prefab size on disk. Throws PrefabLoadException if not found. |
| savePrefab(path, prefab, overwrite) | void | O(N) | Saves a prefab to an absolute path and invalidates its cache entry. Throws PrefabSaveException on I/O error or if file exists and overwrite is false. |
| getServerPrefab(key) | BlockSelection | O(N) | Convenience method to load a prefab from the server's root prefabs directory. |
| getAssetPrefabFromAnyPack(key) | BlockSelection | O(P*D) | Searches all registered AssetPacks for a prefab by key, returning the first one found. P is the number of packs, D is disk access cost. |
| getPrefabDir(dir) | Map | O(F*N) | Loads all valid prefabs within a given directory. F is the number of files, N is the cost to load each prefab. |

## Integration Patterns

### Standard Usage

To load a prefab, first retrieve the singleton instance and then call the appropriate accessor method based on the desired source. The system handles caching and deserialization transparently.

```java
// Standard pattern for loading a server-level prefab
PrefabStore store = PrefabStore.get();
try {
    BlockSelection housePrefab = store.getServerPrefab("structures/village/house_01.prefab.json");
    // ... use the prefab for world modification
} catch (PrefabLoadException e) {
    // Handle cases where the prefab does not exist
}
```

### Anti-Patterns (Do NOT do this)

-   **Manual Path Concatenation:** Do not build file paths manually. The location of asset or world directories can change. Always use helper methods like getServerPrefabsPath or getAssetPrefabsPathForPack to resolve the correct base directory.
-   **External File Modification:** Modifying or deleting a .prefab.json file on disk while the server is running will lead to data inconsistency. The PrefabStore will continue to serve the stale, cached version of the prefab until the server is restarted or the cache is programmatically invalidated via a save operation.
-   **Ignoring Exceptions:** A PrefabLoadException indicates a critical failure, such as a missing file or corrupted data. This exception must be caught and handled appropriately to prevent cascading failures in systems that depend on the prefab, like world generation or structure placement.

## Data Pipeline

The data flow for a standard read operation demonstrates the central role of the cache in the system's design.

> **Flow: Read Operation**
>
> External System Call (e.g., World Generator)
> -> `PrefabStore.get().getPrefab(path)`
> -> Atomic lookup in PREFAB_CACHE
> -> **Cache Hit:** Return cached BlockSelection object immediately.
> -> **Cache Miss:**
>    1.  File System Read (from `path`)
>    2.  `BsonUtil.readDocument()` deserializes raw bytes to BSON
>    3.  `SelectionPrefabSerializer.deserialize()` converts BSON to BlockSelection object
>    4.  BlockSelection object is inserted into PREFAB_CACHE
>    5.  Return new BlockSelection object to caller

