---
description: Architectural reference for BlockTypeAssetMap
---

# BlockTypeAssetMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Stateful Component

## Definition
```java
// Signature
public class BlockTypeAssetMap<K, T extends JsonAssetWithMap<K, BlockTypeAssetMap<K, T>>> extends AssetMapWithIndexes<K, T> {
```

## Architecture & Concepts

The BlockTypeAssetMap is a high-performance, thread-safe data structure that serves as a critical translation layer within the Hytale Asset Store. Its primary function is to map a human-readable asset key, such as a block name, to a dense, zero-based integer index. This index is then used for near-instantaneous O(1) lookups into a contiguous array of loaded asset objects.

This two-stage lookup pattern is fundamental to engine performance. Systems that operate on large volumes of assets per frame, such as the world renderer or physics engine, cannot tolerate the overhead of repeated hash map lookups using complex keys. By resolving the key to a simple integer ID once, subsequent accesses become trivial array lookups.

The map is specifically designed for a "write-infrequently, read-many" workload. Asset loading and unloading (write operations) are heavyweight, synchronized events that rebuild the internal indexes. In contrast, asset retrieval (read operations) are extremely lightweight and highly optimized for massive concurrency using stamped optimistic locking.

Internally, it leverages the **fastutil** library for memory-efficient primitive collections and a case-insensitive hashing strategy, ensuring consistent asset key resolution.

## Lifecycle & Ownership

-   **Creation:** A BlockTypeAssetMap is not intended for direct instantiation by client code. It is created and managed by a higher-level service, typically the central AssetStore, during the engine's bootstrap sequence. Its behavior is configured via a constructor-injected array provider function, which defines the concrete asset type it will manage.
-   **Scope:** The instance persists for the entire lifecycle of its parent AssetStore, which generally corresponds to a full client or server session. It is a long-lived object that accumulates the state of all loaded assets of its type.
-   **Destruction:** The map is marked for garbage collection when the parent AssetStore is shut down. The `clear` method is invoked during asset hot-reloading or shutdown to nullify all references to asset objects and reset internal structures, preventing memory leaks.

## Internal State & Concurrency

-   **State:** This class is highly mutable and stateful. Its core state is composed of two primary data structures:
    1.  `keyToIndex`: A map from an asset key of type K to a primitive integer index.
    2.  `array`: A dense array of asset objects of type T, directly addressable by the integer index.
    The state is modified exclusively through the `putAll`, `remove`, and `clear` methods, which are triggered by asset loading and unloading events.

-   **Thread Safety:** The class is engineered to be thread-safe, but with a significant performance asymmetry between read and write operations.
    -   **Read Path:** Reading an asset is a two-step, highly concurrent process. The `getIndex` method uses a `StampedLock` to perform an optimistic read without any initial locking. A full read lock is only acquired as a fallback if a concurrent write operation was detected. This design heavily favors read-heavy workloads, minimizing contention between threads that are merely querying asset data.
    -   **Write Path:** All write operations (`putAll`, `remove`) are heavyweight, exclusive events. They acquire a full write lock on the `keyToIndex` map and a separate reentrant lock on the `array`. This serializes all modifications to the map, blocking any concurrent reads or writes until the operation is complete.

    **WARNING:** Frequent write operations will cause severe lock contention and performance degradation due to array resizing and copying. This component is not designed for high-frequency state changes.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIndex(K key) | int | O(1) | Resolves an asset key to its integer index. Returns MIN_VALUE if not found. |
| getAsset(int index) | T | O(1) | Retrieves the asset object for a given integer index. Returns null if out of bounds. |
| putAll(...) | void | O(N+M) | Heavyweight operation to add or update a batch of assets. Resizes internal arrays. |
| remove(Set<K> keys) | Set<K> | O(N+M) | Heavyweight operation to remove a batch of assets. May resize internal arrays. |
| clear() | void | O(1) | Removes all assets and resets the internal state. |
| getSubKeys(K key) | ObjectSet<K> | O(1) | Returns a set of related keys, typically for assets with variants. |

## Integration Patterns

### Standard Usage

The correct usage pattern is a two-step lookup. Game systems should resolve the asset key to an integer index once and cache the index for subsequent use within a frame or a short-lived process.

```java
// Assume 'assetStore' provides access to the map
BlockTypeAssetMap<String, BlockAsset> blockMap = assetStore.getBlockMap();

// 1. Resolve the key to an integer index once
int stoneBlockId = blockMap.getIndex("hytale:stone");

// WARNING: Check for invalid index before use
if (stoneBlockId != Integer.MIN_VALUE) {
    // 2. Use the fast integer index for repeated lookups
    BlockAsset stoneAsset = blockMap.getAsset(stoneBlockId);
    // ... use the asset in the game loop or renderer
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new BlockTypeAssetMap()`. The component is tightly coupled to the AssetStore's lifecycle and must be retrieved from the appropriate service registry.
-   **Per-Frame Key Lookups:** Do not call `getIndex("key")` repeatedly for the same asset within a tight loop. Resolve the key to an integer once and reuse the integer.
-   **Index Caching Across Reloads:** Do not store an asset's integer index in a persistent or long-lived component. Asset hot-reloading can invalidate all existing indexes, causing a stored index to point to a different asset or an out-of-bounds location. Always re-resolve keys after an asset reload event.
-   **High-Frequency Writes:** Do not use this map for data that changes frequently. The locking and array-copying overhead of write operations is substantial and will stall reader threads.

## Data Pipeline

The BlockTypeAssetMap sits at the intersection of the asset loading system and real-time game systems. It consumes processed data from the former and provides high-speed access to the latter.

> Flow (Asset Loading):
> Asset File -> AssetCodec (Deserialization) -> `Map<K, T>` -> **BlockTypeAssetMap.putAll** -> Internal State (Index Map & Asset Array)

> Flow (Asset Retrieval):
> Game System (with key K) -> **BlockTypeAssetMap.getIndex** -> `int` -> **BlockTypeAssetMap.getAsset** -> Asset Object (T) -> Game System Logic

