---
description: Architectural reference for DefaultAssetMap, the core implementation for storing and managing versioned game assets.
---

# DefaultAssetMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Stateful Component

## Definition
```java
// Signature
public class DefaultAssetMap<K, T extends JsonAsset<K>> extends AssetMap<K, T> {
```

## Architecture & Concepts
The DefaultAssetMap is the canonical, in-memory database for a single category of game assets (e.g., textures, models, sounds). It is not merely a key-value store; its primary architectural function is to manage the **asset override system**, a critical feature for supporting resource packs and mods.

Each asset key does not map to a single asset, but to a *chain* of assets from different packs. The last asset added to this chain is considered the "winner" and becomes the effective asset returned by standard lookups. This allows a resource pack with a higher priority to override a default Hytale asset without replacing the original file.

The implementation heavily leverages high-performance `fastutil` collections and a sophisticated `StampedLock` to ensure high-throughput, thread-safe access in a concurrent environment, where the game thread might be reading assets while a background thread is loading or unloading a resource pack.

### Lifecycle & Ownership
- **Creation:** An instance of DefaultAssetMap is created by the central AssetStore service during engine initialization. A distinct instance is typically created for each major asset type managed by the store.
- **Scope:** The object's lifecycle is bound to the AssetStore. It persists for the entire application session, accumulating state as asset packs are loaded.
- **Destruction:** The map is cleared via the `clear` method when the AssetStore performs a full reload or during application shutdown. The Java garbage collector reclaims the object itself only when the AssetStore is destroyed.

## Internal State & Concurrency
- **State:** The internal state is highly mutable and complex, serving as a live cache of all loaded assets of a specific type. Key internal structures include:
    - **assetMap:** A map from an asset key to the *final, effective* asset object. This is the "winning" asset after all overrides are applied.
    - **assetChainMap:** The core of the override system. Maps an asset key to an array of AssetRef objects, where each reference represents the asset as defined in a specific resource pack. The order of this array determines override priority.
    - **tagStorage:** An optimized, integer-indexed map for retrieving sets of asset keys associated with a specific tag, enabling fast, category-based lookups.
    - **pathToKeyMap:** A reverse mapping from a file system path to the asset keys defined within that file.

- **Thread Safety:** This class is designed to be thread-safe.
    - Concurrency is managed by a `StampedLock`. Read operations like `getAsset` employ an **optimistic read** pattern. They first attempt a read without a lock and then validate it. If a write occurred concurrently, the read is retried under a full read lock. This pattern significantly reduces contention in read-heavy scenarios.
    - Write operations such as `putAll` and `remove` acquire an exclusive write lock, ensuring that modifications to the asset chains and primary maps are atomic and fully visible to subsequent reads.

    **WARNING:** While individual method calls are thread-safe, sequences of calls are not atomic. For example, retrieving an asset and then its children in two separate calls does not guarantee that the children list is consistent with the parent asset if a resource pack was modified between the calls.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsset(K key) | T | O(1) | Retrieves the final, effective asset for a given key after all overrides. Uses an optimistic read lock. |
| getAsset(String packKey, K key) | T | O(N) | Retrieves a specific version of an asset from the override chain, as it was before being overridden by the specified pack. |
| putAll(...) | void | O(M) | The primary ingestion method. Atomically adds a batch of M assets from a pack, updating the override chains and effective asset map. Acquires a full write lock. |
| remove(Set<K> keys) | Set<K> | O(M) | Removes all versions of M assets from all packs. This is a destructive operation used for full asset removal. Acquires a full write lock. |

## Integration Patterns

### Standard Usage
The DefaultAssetMap is not intended for direct use by game logic. It is an internal component of the AssetStore. Game systems should always interact with the higher-level AssetStore or AssetManager, which abstracts away the specific map instance.

```java
// A game system (e.g., Renderer) gets the central store
AssetStore assetStore = engine.getAssetStore();

// The store internally delegates the request to the correct DefaultAssetMap
// For example, the one holding Model assets.
Model playerModel = assetStore.getAsset(Model.class, "hytale:player");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new DefaultAssetMap()`. The lifecycle is strictly managed by the AssetStore to ensure it is correctly registered and populated.
- **External Modification:** Do not attempt to get a reference to the internal collections and modify them. Bypassing the `putAll` or `remove` methods will break the locking mechanism and corrupt the state of the override chains.
- **Unsafe Iteration:** Do not iterate over the map returned by `getAssetMap` while assuming it will not change. A resource pack could be loaded or unloaded on another thread, causing a ConcurrentModificationException.

## Data Pipeline

The component serves two primary data flows: ingestion (loading) and retrieval (serving).

**Ingestion Flow:**
> Flow:
> Asset File on Disk -> AssetCodec -> **DefaultAssetMap.putAll()** -> Internal Override Chains Updated -> Final Asset Cached

**Retrieval Flow:**
> Flow:
> Game System Request -> AssetStore -> **DefaultAssetMap.getAsset()** -> Returns Cached Final Asset

