---
description: Architectural reference for IndexedLookupTableAssetMap
---

# IndexedLookupTableAssetMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Component / Data Structure

## Definition
```java
// Signature
public class IndexedLookupTableAssetMap<K, T extends JsonAssetWithMap<K, IndexedLookupTableAssetMap<K, T>>> extends AssetMapWithIndexes<K, T> {
```

## Architecture & Concepts
The IndexedLookupTableAssetMap is a high-performance, thread-safe data structure designed for storing and retrieving game assets. Its primary architectural purpose is to provide extremely fast asset lookups by mapping a high-level key, such as a string name, to a compact integer index. This index is then used for a direct, O(1) lookup into a contiguous array.

This pattern is a critical optimization in game engines, where asset retrieval happens thousands of times per frame. It trades the flexibility and higher constant-time overhead of a standard HashMap for the raw speed and superior cache-locality of an array access.

Key architectural features include:
-   **Key-to-Index Mapping:** It maintains an internal map (`keyToIndex`) that translates a generic asset key `K` into a unique, sequentially generated integer. This map uses the `fastutil` library and a `CaseInsensitiveHashStrategy` to ensure both performance and usability for string-based keys.
-   **Dense Array Storage:** Assets are stored in a tightly packed array (`array`). Accessing an asset via its integer index is a direct memory operation, making it significantly faster than hash-based lookups.
-   **Dynamic Array Resizing:** The internal array automatically grows as new assets are added with higher indices, but it only shrinks when assets are removed from the end of the array. This strategy prioritizes insertion speed over memory reclamation during gameplay.
-   **Advanced Concurrency Control:** The class employs a sophisticated locking strategy using both `StampedLock` and `ReentrantLock` to allow for highly concurrent reads while ensuring safety during write operations. The `StampedLock` on the key-to-index map uses an optimistic read pattern, which avoids locking overhead in the common case where no writes are occurring.

This class is not intended for direct use by most game logic, but rather as a foundational component within a higher-level AssetManager or AssetStore service.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by a parent asset management system that requires an indexed lookup strategy. The caller must provide an `IntFunction<T[]>` (typically a constructor reference like `MyAssetType[]::new`) which the map uses to create and resize its internal storage array.
-   **Scope:** The lifecycle of an IndexedLookupTableAssetMap is tied to the collection of assets it manages. It typically persists for the duration of a game session or until a full asset reload is triggered (e.g., changing a resource pack).
-   **Destruction:** The object is eligible for garbage collection when its owning AssetManager is discarded. The `clear` method can be invoked to release all contained assets and reset its internal state without destroying the map object itself.

## Internal State & Concurrency
-   **State:** This class is highly mutable. It manages a collection of assets, the mapping of their keys to integer indices, and metadata about the collection. Its primary state consists of the `keyToIndex` map and the `array` of assets.
-   **Thread Safety:** This class is explicitly designed to be thread-safe and can be safely accessed from multiple threads.
    -   Read operations (`getIndex`, `getAsset`) are highly optimized for concurrency. `getIndex` uses a `StampedLock` with an optimistic read, which is extremely fast under low write contention.
    -   Write operations (`putAll`, `remove`, `clear`) acquire exclusive write locks on both the index map and the asset array. This ensures atomicity and prevents the internal state from becoming inconsistent.
    -   **WARNING:** The internal locking is fine-grained and complex. Callers must **NOT** attempt to perform external synchronization on instances of this class, as it can easily lead to deadlocks with the internal lock mechanisms.

## API Surface
The public API is designed for querying the asset collection. Modification is handled through the protected `AssetMapWithIndexes` contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIndex(K key) | int | O(1) avg | Retrieves the integer index for the given asset key. Returns MIN_VALUE if not found. |
| getAsset(int index) | T | O(1) | Retrieves the asset at the specified integer index. Returns null if the index is out of bounds. |
| getNextIndex() | int | O(1) | Returns the value of the next index that will be assigned to a new asset. |

## Integration Patterns

### Standard Usage
This class is an implementation detail of the asset system. A developer would typically interact with a service that uses this map internally. The conceptual flow is to first resolve a key to an index, then use that index for fast, repeated access.

```java
// Conceptual example of how a parent service would use this map
// Note: Direct interaction is not the standard pattern.

// 1. The service retrieves the integer ID for an asset name once.
int stoneBlockId = blockAssetMap.getIndex("hytale:stone");

// 2. The ID is then used for all subsequent, high-performance lookups.
// This is much faster than looking up by string every time.
BlockAsset stoneBlock = blockAssetMap.getAsset(stoneBlockId);
renderer.draw(stoneBlock.getTexture());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create instances of this class using `new` in general game logic. It should be configured and managed exclusively by the core asset loading systems to ensure it is correctly populated and integrated into the asset hot-reloading pipeline.
-   **State Assumption After Write:** Do not perform a read immediately after a write from a different thread without a memory barrier. While the class is thread-safe, the visibility of a write to other threads is subject to standard Java Memory Model guarantees. It is safer to interact with the map from a single, designated update thread or use higher-level synchronization primitives if required.
-   **Iteration During Modification:** Do not attempt to iterate over the map's contents while another thread is modifying it. The class does not provide fail-fast iterators, and this can lead to unpredictable behavior or runtime exceptions.

## Data Pipeline
The IndexedLookupTableAssetMap sits at the core of the asset hydration process, transforming deserialized asset data into a runtime-optimized queryable structure.

> **Ingestion Flow:**
> Asset File (JSON) -> AssetCodec (Deserialization) -> `AssetMap.putAll` -> **IndexedLookupTableAssetMap** (Assigns integer index, inserts into array)

> **Retrieval Flow:**
> Game System (e.g., Renderer) -> `getAsset("key")` -> **IndexedLookupTableAssetMap** (`getIndex` -> `getAsset(index)`) -> Asset Object returned to Game System

