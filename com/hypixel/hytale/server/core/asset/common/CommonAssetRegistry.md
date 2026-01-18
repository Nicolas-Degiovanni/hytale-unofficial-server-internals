---
description: Architectural reference for CommonAssetRegistry
---

# CommonAssetRegistry

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Utility

## Definition
```java
// Signature
public class CommonAssetRegistry {
```

## Architecture & Concepts
The CommonAssetRegistry is a static, in-memory database that serves as the single source of truth for all discoverable game assets on the server. It is a foundational component of the asset loading system, responsible for indexing assets from various sources (e.g., base game files, mods, resource packs) and resolving conflicts between them.

Its core architectural function is to manage **asset overriding**. The system is designed around a "last-writer-wins" principle. When multiple asset packs provide a file with the same canonical name (e.g., *models/creatures/skeleton.json*), the registry ensures that only the version from the most recently loaded pack is considered "active". This is fundamental for allowing mods and resource packs to replace default game assets.

To achieve this, the registry maintains two primary indexes:
1.  **Name-Based Index** (*assetByNameMap*): A map from an asset's string name to a list of all assets with that name, ordered by load priority. The last element in this list is always the active asset.
2.  **Hash-Based Index** (*assetByHashMap*): A map from an asset's content hash to a list of assets that share that hash. This is primarily used for data deduplication, consistency checks, and identifying identical assets that may have different names.

This dual-index approach provides both fast, name-based lookups for game logic and powerful, content-based lookups for diagnostics and optimization.

### Lifecycle & Ownership
-   **Creation:** The CommonAssetRegistry is a static utility class. Its state is initialized in static memory when the class is first loaded by the JVM ClassLoader. There are no instances of this class.
-   **Scope:** The registry's state is global and application-scoped. It persists for the entire lifetime of the server process.
-   **Destruction:** The internal maps are only cleared upon an explicit call to `clearAllAssets`. This typically occurs during a server shutdown sequence or a full, controlled reload of all game assets. Otherwise, the memory is reclaimed when the JVM process terminates.

## Internal State & Concurrency
-   **State:** The class's state is entirely mutable, consisting of static maps that are populated and modified as asset packs are loaded and unloaded. It is a central, shared, and highly dynamic state.
-   **Thread Safety:** This class is designed to be thread-safe and is a primary example of engineering for concurrent data access in the engine.
    -   The main index maps, *assetByNameMap* and *assetByHashMap*, are implemented as **ConcurrentHashMap**. This provides high-performance, thread-safe access at the map level without broad locking.
    -   The values within these maps are lists of type **CopyOnWriteArrayList**. This is a critical design choice. It ensures that read operations (like asset lookups) are extremely fast and lock-free, as they operate on an immutable snapshot of the list. Write operations (adding or removing an asset) are more expensive as they involve creating a new copy of the underlying array.
    -   **WARNING:** This design heavily favors a "write-rarely, read-often" workload, which is typical for asset registries (i.e., assets are loaded once at startup, then looked up frequently during gameplay).

## API Surface
The public API consists entirely of static methods for querying and manipulating the global asset state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addCommonAsset(pack, asset) | AddCommonAssetResult | O(N) | Registers an asset. N is the number of existing overrides for the asset name. This is the primary ingestion method. |
| removeCommonAssetByName(pack, name) | BooleanObjectPair | O(N) | Removes an asset by its pack and name. N is the number of overrides. Triggers re-evaluation of the active asset. |
| getByName(name) | CommonAsset | O(1) | Retrieves the currently active asset for a given name. This is the most common, high-performance lookup operation. |
| getByHash(hash) | CommonAsset | O(1) | Retrieves an asset by its content hash. Useful for diagnostics and content verification. |
| clearAllAssets() | void | O(M) | Wipes the entire registry. M is the total number of registered assets. **WARNING:** This is a destructive global operation. |
| getAllAssets() | Collection | O(1) | Returns an unmodifiable, snapshot view of all registered asset lists. |
| getDuplicatedAssets() | Map | O(M) | Scans the hash map to find all assets with identical content but potentially different names. This is a diagnostic tool. |

## Integration Patterns

### Standard Usage
The registry is manipulated by asset loaders during the server bootstrap or hot-reload process. Game systems then query it to resolve asset paths to concrete data.

```java
// During asset pack loading
AssetPackLoader loader = ...;
for (CommonAsset asset : loader.discoverAssets()) {
    CommonAssetRegistry.addCommonAsset(loader.getPackName(), asset);
}

// During gameplay
GameSystem system = ...;
CommonAsset stoneTexture = CommonAssetRegistry.getByName("textures/world/stone.png");
if (stoneTexture != null) {
    system.use(stoneTexture);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Assumption During Loading:** Do not query the registry for an asset and assume its state is final while other threads may still be loading asset packs. The "active" asset can change multiple times. Systems should wait for a global "AssetLoadingComplete" event before caching results from `getByName`.
-   **Modification of Returned Collections:** The collection returned by `getAllAssets` is unmodifiable. Attempting to modify it will result in an `UnsupportedOperationException`.
-   **Frequent Writes:** Do not design a system that calls `addCommonAsset` or `removeCommonAssetByName` frequently during the main game loop. The use of `CopyOnWriteArrayList` makes these operations expensive and unsuitable for high-frequency execution.

## Data Pipeline
The CommonAssetRegistry acts as a central sink and source for asset metadata. It does not process data itself but rather organizes it for consumption by other systems.

> **Ingestion Flow:**
> AssetPack on Disk -> AssetPackLoader -> `CommonAsset` instance -> **CommonAssetRegistry.addCommonAsset()** -> In-Memory Maps

> **Retrieval Flow:**
> Game Logic Request (e.g., "stone.png") -> **CommonAssetRegistry.getByName()** -> `CommonAsset` instance -> Renderer / Sound Engine / etc.

