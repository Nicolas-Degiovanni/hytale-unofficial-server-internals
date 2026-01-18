---
description: Architectural reference for AssetStore
---

# AssetStore

**Package:** com.hypixel.hytale.assetstore
**Type:** Managed Component

## Definition
```java
// Signature
public abstract class AssetStore<K, T extends JsonAssetWithMap<K, M>, M extends AssetMap<K, T>> {
```

## Architecture & Concepts
The AssetStore is the canonical repository and lifecycle manager for a single category of game assets, such as models, textures, or sound definitions. It is a foundational component of the engine's resource management system, orchestrated by the central AssetRegistry.

Each concrete implementation of AssetStore is responsible for one type of asset, defined by its generic parameters. It acts as the bridge between raw data on the file system and the structured, validated, and interconnected asset objects used by the game at runtime.

Key architectural concepts include:

-   **Type Specialization:** The engine maintains one AssetStore instance per asset type. This provides a type-safe, centralized point of access and modification for all assets of that kind.
-   **Dependency Management:** AssetStores declare explicit loading dependencies using *loadsAfter* and *loadsBefore*. The AssetRegistry resolves these declarations into a Directed Acyclic Graph (DAG) to ensure that assets are loaded and available in the correct order. For example, a Model asset store must load *after* its corresponding Texture asset store.
-   **Asset Inheritance:** The system supports a powerful inheritance model where an asset can be defined as a child of another. The AssetStore resolves this relationship during decoding, merging the child's data over the parent's to create the final object. This is used extensively to reduce data duplication.
-   **Contained & Child Assets:** The architecture distinguishes between two forms of asset composition. *Contained Assets* are defined inline within a parent asset's file and are managed by the parent's lifecycle. *Child Assets* are standard, standalone assets that have a logical parent-child relationship established. The AssetStore manages both, tracking these relationships in its *childAssetsMap*.
-   **Asynchronous Loading:** Asset decoding is a computationally intensive task. The AssetStore parallelizes this work heavily using a pool of worker threads and CompletableFuture, significantly reducing initial load times.

### Lifecycle & Ownership
-   **Creation:** AssetStore instances are not created directly. They are instantiated and configured exclusively by the AssetRegistry during the engine's initial bootstrap sequence. Each store is built using a fluent Builder pattern that defines its path, codec, dependencies, and other behaviors.
-   **Scope:** An AssetStore is session-scoped. It persists for the entire lifetime of the application, whether it is a client or a server. Its internal state, the *assetMap*, represents the complete, live catalog of its managed asset type.
-   **Destruction:** The AssetStore and its cached assets are garbage collected when the AssetRegistry is shut down as part of the application's termination process.

## Internal State & Concurrency
-   **State:** The AssetStore is a stateful, mutable component. Its primary state is the *assetMap*, which caches all loaded assets. It also maintains mappings from asset keys to file paths, parent-child relationships, and dependencies on other stores.

-   **Thread Safety:** This class is designed for concurrent access but is **not** inherently thread-safe. Concurrency is managed by a single, global write lock, **AssetRegistry.ASSET_LOCK**.
    -   All public methods that mutate the state of the asset system, such as *loadAssets* and *removeAssets*, acquire this lock internally.
    -   The internal decoding process is highly parallel, but the final commit of decoded assets into the *assetMap* is synchronized under this global lock.
    -   **WARNING:** Any external system that directly reads from the *assetMap* (retrieved via *getAssetMap*) while asset loading might be in progress **must** manually acquire the corresponding read lock from AssetRegistry.ASSET_LOCK to prevent race conditions.

## API Surface
The public API focuses on high-level load and remove operations. Direct manipulation of assets is typically done through the retrieved AssetMap.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadAssetsFromDirectory(packKey, assetsPath) | AssetLoadResult | O(N * D) | Scans a directory for asset files and orchestrates their decoding and loading. N is number of files, D is decode complexity. |
| loadAssets(packKey, assets) | AssetLoadResult | O(N) | Loads a list of pre-instantiated asset objects into the store. Primarily used for dynamic or generated assets. |
| removeAssets(keys) | Set | O(N) | Removes a collection of assets by their keys and recursively removes their children. Fires a RemovedAssetsEvent. |
| removeAssetPack(name) | void | O(N) | Removes all assets associated with a specific asset pack. |
| getAssetMap() | M | O(1) | Returns the underlying AssetMap that serves as the live cache for all loaded assets of this type. |
| validate(key, results, extraInfo) | void | O(1) | A hook for the engine's validation framework to verify that a given asset key exists within this store. |

## Integration Patterns

### Standard Usage
An AssetStore should never be instantiated directly. It must be retrieved from the central AssetRegistry, after which its AssetMap can be used to access assets.

```java
// How a developer should normally use this
AssetStore<String, Model, ?> modelStore = AssetRegistry.getAssetStore(Model.class);
AssetMap<String, Model> modelMap = modelStore.getAssetMap();
Model playerModel = modelMap.getAsset("hytale:player");

if (playerModel != null) {
    // Use the asset
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The class is abstract and managed by the AssetRegistry. Attempting to construct it manually will bypass dependency resolution and global state management, leading to system instability.
-   **Unsynchronized Map Access:** Reading from the result of *getAssetMap* from a separate thread without acquiring the global *AssetRegistry.ASSET_LOCK.readLock()* is a severe race condition. The asset map can be modified at any time by the asset loading thread.
-   **Dynamic Dependency Injection:** The *injectLoadsAfter* method is deprecated and should not be used. Asset loading dependencies must be declared statically during the bootstrap phase. Modifying them at runtime can break the loading DAG and cause unpredictable behavior.

## Data Pipeline
The AssetStore is a critical stage in the data pipeline that transforms raw files into usable game objects.

> Flow:
> File System (.json) -> I/O Read -> **AssetStore** (Orchestrates parallel decoding via AssetCodec) -> **AssetStore** (Caches result in AssetMap) -> Event Bus (LoadedAssetsEvent) -> Game Systems (Renderer, AI, etc.)

---

