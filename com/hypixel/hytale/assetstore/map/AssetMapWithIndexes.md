---
description: Architectural reference for AssetMapWithIndexes
---

# AssetMapWithIndexes

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Stateful Framework Component

## Definition
```java
// Signature
public abstract class AssetMapWithIndexes<K, T extends JsonAsset<K>> extends DefaultAssetMap<K, T> {
```

## Architecture & Concepts
AssetMapWithIndexes is an abstract base class that enhances a standard asset map with a secondary, tag-based indexing system. Its primary architectural function is to enable extremely fast, O(1) lookups of asset collections based on integer-encoded tags. This component acts as a specialized, in-memory search index that sits alongside the primary key-value asset storage.

This class is fundamental to systems that need to query assets by their properties rather than their unique keys. For example, the rendering engine might query for all assets tagged as "emissive", or a gameplay system might request all items tagged as "crafting_ingredient". By pre-computing and caching these relationships, it decouples the game logic's query needs from the underlying asset storage structure, preventing expensive, full-map iterations.

The core design revolves around two parallel hash maps: one for mutable index construction and another that exposes a thread-safe, unmodifiable view for external consumers.

### Lifecycle & Ownership
- **Creation:** This is an abstract class and cannot be instantiated directly. Concrete subclasses (e.g., a theoretical ModelAssetMap) are instantiated by the AssetStore service during the asset loading phase.
- **Scope:** An instance of a subclass persists for the entire duration that its corresponding asset type is loaded in memory. Its lifecycle is directly coupled to the lifecycle of the parent AssetStore.
- **Destruction:** The instance is marked for garbage collection when the AssetStore performs a full clear, such as during a client shutdown or a hot-reload of all assets. The `clear` method ensures that all internal index maps are purged to release memory.

## Internal State & Concurrency
- **State:** The class maintains a significant, mutable internal state consisting of two maps:
    - **indexedTagStorage:** A concurrent hash map that stores the primary, mutable mapping from a tag's integer ID to a set of asset indexes. This map is actively modified as assets are added.
    - **unmodifiableIndexedTagStorage:** A parallel concurrent hash map that stores unmodifiable wrappers around the integer sets from indexedTagStorage. This provides a safe, read-only view for external queries.

- **Thread Safety:** This class is designed to be thread-safe for both reads and writes.
    - Writes (index building) are managed via `Int2ObjectConcurrentHashMap`, which provides high-throughput, thread-safe operations. The `computeIfAbsent` pattern is used to atomically create new index sets, preventing race conditions.
    - Reads via `getIndexesForTag` are inherently safe as they operate on the `unmodifiableIndexedTagStorage` map and return an unmodifiable set, preventing client code from corrupting the index.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIndexesForTag(int index) | IntSet | O(1) | Retrieves an unmodifiable set of asset indexes associated with a tag. Returns an empty set if the tag is not found. This is the primary query method. |
| putAssetTag(K key, int index, int tag) | void | O(1) amortized | **Internal.** Adds a single asset index to a specific tag's index set. This is the core mutation method used during index construction. |
| requireReplaceOnRemove() | boolean | O(1) | **CRITICAL.** Returns true, signaling to the owner of this map that assets cannot be simply removed. A removal must be followed by a replacement (e.g., with a null or placeholder entry) to maintain the integrity of other asset indexes. |
| clear() | void | O(N) | Wipes all asset mappings and all tag indexes. Called during major lifecycle events like asset reloads. |

## Integration Patterns

### Standard Usage
This class is not used directly. Game developers interact with a higher-level service (e.g., an AssetManager or a specialized query service) that uses this map's indexing capabilities under the hood.

```java
// Hypothetical AssetQueryService using the map
public class AssetQueryService {
    // This would be a concrete implementation, e.g., ModelAssetMap
    private AssetMapWithIndexes<String, ModelAsset> modelMap;

    public Set<ModelAsset> findModelsByTag(String tagName) {
        int tagId = TagRegistry.getId(tagName);
        IntSet indexes = modelMap.getIndexesForTag(tagId);

        // Convert indexes back to assets
        return indexes.stream()
                      .map(modelMap::getAssetByIndex)
                      .collect(Collectors.toSet());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring `requireReplaceOnRemove`:** The contract requires that removed assets are replaced, not just deleted from the underlying storage. Failure to do so will lead to index corruption and `IndexOutOfBoundsException` because other asset indexes will not be shifted correctly. The owning data structure must handle this.
- **Attempting to Modify Returned Sets:** The `IntSet` returned by `getIndexesForTag` is unmodifiable. Any attempt to add or remove elements will result in an `UnsupportedOperationException`. Do not cast or otherwise attempt to bypass this protection.

## Data Pipeline
The data flow for this component is centered on the asset loading and indexing process. It transforms metadata embedded within an asset into a fast, queryable index.

> Flow:
> Asset File -> AssetCodec -> AssetExtraInfo.Data (Tags Extracted) -> **AssetMapWithIndexes.putAssetTag** -> Internal Index Built -> Game System Query -> **AssetMapWithIndexes.getIndexesForTag** -> Set of Asset Indexes

