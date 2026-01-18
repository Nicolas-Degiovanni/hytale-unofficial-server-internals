---
description: Architectural reference for AssetMap
---

# AssetMap

**Package:** com.hypixel.hytale.assetstore
**Type:** Base Class / Contract

## Definition
```java
// Signature
public abstract class AssetMap<K, T extends JsonAsset<K>> {
```

## Architecture & Concepts
The AssetMap is an abstract base class that defines the contract for a specialized, in-memory repository of a specific type of game asset. It serves as a foundational component within the Hytale Asset Subsystem, providing a structured and queryable interface for a collection of assets loaded from disk.

Unlike a generic Java Map, an AssetMap is a highly specialized data structure designed to understand the metadata and relationships inherent to game assets. It maintains not only the primary key-to-asset mapping but also ancillary data such as:
- The source file path for each asset.
- The asset pack an asset belongs to.
- Hierarchical parent-child relationships between assets.
- Tag-based indexing for efficient group lookups.

Concrete implementations of AssetMap (e.g., for models, sounds, or textures) are managed by a higher-level service, likely an AssetManager. This design decouples the core game logic from the complexities of asset storage and retrieval, allowing systems to request assets by a logical key without needing knowledge of the underlying file system or asset pack structure.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of AssetMap are not instantiated directly by game logic. They are created and populated by the core asset loading system during the engine's initialization phase or during a hot-reload event (e.g., changing a resource pack).
- **Scope:** An AssetMap instance persists for as long as its asset collection is considered valid. Typically, this means it lives for the entire client or server session. Its lifetime is strictly managed by the overarching AssetManager.
- **Destruction:** The map is cleared via the protected `clear` method when the AssetManager orchestrates a full asset reload. Individual assets or asset packs can be unloaded dynamically using the `remove` methods, which are also invoked exclusively by the asset management system.

## Internal State & Concurrency
- **State:** An AssetMap is fundamentally a stateful and mutable component. It holds several internal collections that cache loaded assets and their associated metadata. The data it contains is a direct reflection of the asset files loaded from disk.
- **Thread Safety:** This abstract class provides no inherent thread safety guarantees. All methods are non-synchronized. It is the responsibility of the concrete implementation or the calling system (the AssetManager) to ensure that concurrent access is properly managed.

**WARNING:** Reading from an AssetMap on the main game thread while it is being modified by a background asset loading thread will lead to undefined behavior, including ConcurrentModificationExceptions and inconsistent game state. All modifications must be synchronized or carefully orchestrated to occur during safe points in the game loop.

## API Surface
The public API is designed for read-only queries by game systems. Mutating operations are protected, intended only for use by the asset loading framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsset(K key) | T | O(1) | Retrieves a single asset by its primary key. Returns null if not found. |
| getPath(K key) | Path | O(1) | Resolves the absolute file system path for a given asset key. |
| getChildren(K key) | Set<K> | O(N) | Fetches the set of keys that are defined as children of the given asset key. |
| getKeysForTag(int tagIndex) | Set<K> | O(N) | Returns all asset keys associated with a pre-computed integer tag index. |
| getAssetCount() | int | O(1) | Returns the total number of assets currently held in the map. |
| getAssetMap() | Map<K, T> | O(1) | **WARNING:** Returns a direct reference to the internal map. Modification is unsafe. |

## Integration Patterns

### Standard Usage
Direct interaction with an AssetMap is uncommon for most game logic. The standard pattern is to use a high-level manager that abstracts away the specific AssetMap implementation. The following example is illustrative of how an engine-level system would use a concrete AssetMap.

```java
// Engine-level code, likely within an AssetManager
// Assume 'modelMap' is a concrete AssetMap<ModelKey, ModelAsset>
ModelKey key = new ModelKey("hytale:player");
ModelAsset playerModel = modelMap.getAsset(key);

if (playerModel != null) {
    // Use the asset
    renderer.submit(playerModel);
}
```

### Anti-Patterns (Do NOT do this)
- **External Modification:** Never modify the collections returned by methods like `getAssetMap` or `getChildren`. Doing so will corrupt the internal state of the asset system and is not supported.
- **Assuming Persistence:** Do not hold onto direct references to assets retrieved from the map across major state changes (e.g., loading screens). The underlying AssetMap may be cleared and re-populated during a resource pack reload, invalidating the reference. Always re-query for the asset when needed.
- **Unmanaged Instantiation:** Never create an instance of a concrete AssetMap with `new`. They are exclusively managed by the engine's asset loading service.

## Data Pipeline
AssetMap acts as a destination and a source within the broader asset pipeline. It does not process data itself but rather stores the results of the loading and decoding process.

> **Ingress Flow (Asset Loading):**
> Asset File on Disk -> AssetCodec Decoder -> **AssetMap.putAll** -> Internal Data Structures

> **Egress Flow (Asset Retrieval):**
> Game System Request -> AssetManager -> **AssetMap.getAsset** -> Asset Instance Returned

