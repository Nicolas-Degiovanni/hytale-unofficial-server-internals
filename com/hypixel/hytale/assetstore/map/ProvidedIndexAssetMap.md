---
description: Architectural reference for ProvidedIndexAssetMap
---

# ProvidedIndexAssetMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Stateful Component

## Definition
```java
// Signature
public class ProvidedIndexAssetMap<K, T extends JsonAssetWithMap<K, ProvidedIndexAssetMap<K, T>>> 
    extends AssetMapWithIndexes<K, T> {
```

## Architecture & Concepts

The ProvidedIndexAssetMap is a specialized, high-performance data structure within the Hytale Asset Store framework. Its primary function is to maintain a thread-safe, bidirectional mapping between a case-insensitive asset key (K) and a stable, pre-defined integer index.

Unlike a conventional map that might generate its own indices, this class operates on the principle that the integer index is an intrinsic property of the asset data itself. This index is extracted from the asset object (T) during the loading process via a developer-supplied `indexGetter` function.

This design is critical for engine subsystems that rely on constant-time, integer-based lookups for performance, such as rendering, networking, or save-game serialization. By using pre-authored indices, the engine avoids the overhead of string comparisons and ensures that asset references remain stable across different game sessions and versions. The use of a `CaseInsensitiveHashStrategy` ensures that keys like *hytale:stone* and *hytale:Stone* are treated as identical, preventing duplicate asset entries.

## Lifecycle & Ownership

-   **Creation:** An instance of ProvidedIndexAssetMap is not created directly. It is instantiated by a higher-level asset management service, such as an AssetStore or AssetPackLoader, during the initialization of an asset category. A critical dependency, the `indexGetter` function, must be provided at construction to define how integer indices are extracted from asset objects.
-   **Scope:** The object's lifetime is tightly coupled to the asset collection it manages. It persists as long as the corresponding asset pack or category is considered active in memory. It is not a global singleton; multiple instances can and do exist to manage different types of assets.
-   **Destruction:** The object is eligible for garbage collection when the parent asset manager unloads the associated asset pack. The `clear` method is invoked as part of the unloading process to release all internal mappings and memory.

## Internal State & Concurrency

-   **State:** This class is highly mutable. Its core state is the `keyToIndex` map, which is populated, modified, and cleared as assets are loaded and unloaded by the Asset Store.
-   **Thread Safety:** This class is explicitly designed for concurrent access and is thread-safe. It employs a `StampedLock` to mediate access to its internal map, with a strong optimization for read-heavy workloads.
    -   **Read Operations:** Methods like `getIndex` use an optimistic read pattern. They first attempt to read the value without acquiring a full lock. If no concurrent write occurred during the read, the operation completes with minimal overhead. If a write collision is detected, the operation transparently escalates to a full, more expensive read lock to guarantee correctness.
    -   **Write Operations:** Methods like `putAll`, `remove`, and `clear` acquire an exclusive write lock. This ensures that all modifications to the internal state are atomic and fully visible to subsequent reads from any thread.

## API Surface

The public API is primarily for consumption by the parent Asset Store system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIndex(K key) | int | O(1) | Retrieves the pre-defined integer index for the given asset key. Uses an optimistic lock. |
| getIndexOrDefault(K key, int def) | int | O(1) | Retrieves the index for a key, returning a specified default value if the key is not found. |
| requireReplaceOnRemove() | boolean | O(1) | Returns false, signaling to the Asset Store that removed assets do not need a placeholder. |

## Integration Patterns

### Standard Usage

This class is an internal component of the asset system and should not be used directly by gameplay code. Higher-level managers use it to resolve asset keys to performant integer IDs.

```java
// This code is conceptual and executes within the Asset Store

// 1. The map is configured with a function to get the index from the asset
ToIntBiFunction<String, MyAsset> indexExtractor = (key, asset) -> asset.getNetworkId();
ProvidedIndexAssetMap<String, MyAsset> assetMap = new ProvidedIndexAssetMap<>(indexExtractor);

// 2. During asset loading, the map is populated
assetMap.putAll(packName, codec, loadedAssets, ...);

// 3. Engine systems can now perform fast lookups
int stoneId = assetMap.getIndex("hytale:stone");
// The integer stoneId is now used by rendering, networking, etc.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ProvidedIndexAssetMap()` in gameplay logic. The Asset Store is responsible for its creation and lifecycle.
-   **Incorrect Index Getter:** Supplying an `indexGetter` function that produces non-unique or inconsistent indices for a given set of assets will lead to severe data corruption, asset mis-references, and undefined behavior. The stability of the system relies on the correctness of this function.
-   **External Modification:** Never attempt to access or modify the internal `keyToIndex` map via reflection. Bypassing the `StampedLock` will break thread safety and corrupt the state of the Asset Store.

## Data Pipeline

This component acts as a lookup cache rather than a processing node. Data flows into it during asset loading and is queried during gameplay.

**Ingestion Flow:**
> Asset File on Disk -> AssetCodec Deserializer -> `putAll` Method -> **ProvidedIndexAssetMap** (Internal `keyToIndex` map is populated)

**Query Flow:**
> Engine System (e.g., Renderer) -> AssetManager.getAssetId("key") -> **ProvidedIndexAssetMap**.getIndex("key") -> Returns Integer ID

