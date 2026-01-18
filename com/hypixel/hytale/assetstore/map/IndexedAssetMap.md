---
description: Architectural reference for IndexedAssetMap
---

# IndexedAssetMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Component

## Definition
```java
// Signature
public class IndexedAssetMap<K, T extends JsonAssetWithMap<K, IndexedAssetMap<K, T>>> extends AssetMapWithIndexes<K, T> {
```

## Architecture & Concepts

The IndexedAssetMap is a high-performance, thread-safe data structure that serves as a foundational component within the Hytale asset system. Its primary architectural role is to translate arbitrary, often string-based, asset keys of type K into unique, stable integer identifiers.

This translation is a critical performance optimization. Integer identifiers are significantly more efficient for lookups, comparisons, and storage in contiguous memory structures (like arrays or packed buffers) than complex key objects. Systems like rendering, networking, and entity-component management rely on these integer IDs for fast, cache-friendly access to asset data.

The class is built upon `fastutil` collections for minimal memory overhead and high throughput. The explicit use of a `CaseInsensitiveHashStrategy` indicates its design is tailored for string-like keys where case variations should resolve to the same underlying asset.

At its core, IndexedAssetMap maintains a bidirectional relationship: it stores the mapping from a key K to an integer index, while its parent class, AssetMapWithIndexes, likely uses that integer index to manage the actual asset data T.

## Lifecycle & Ownership

-   **Creation:** An IndexedAssetMap is not a globally shared singleton. It is instantiated by the asset loading pipeline as part of a larger asset container. The generic type constraint suggests that these maps can be nested within other assets, forming a hierarchical or tree-like structure of indexed collections.
-   **Scope:** The lifecycle of an IndexedAssetMap is strictly bound to its parent asset container. It is populated when assets are loaded via the `putAll` method and persists as long as the parent container is held in memory.
-   **Destruction:** The map is eligible for garbage collection when its owning asset container is unloaded and dereferenced. The `clear` method provides a mechanism for explicit state removal, which is essential during asset hot-reloading to prevent stale data.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable, consisting primarily of the `keyToIndex` map which caches the key-to-integer mappings. This state is built incrementally as asset packs are loaded and can be modified by subsequent asset reloads or removals.

-   **Thread Safety:** This class is explicitly designed for high-contention, read-heavy environments and is thread-safe.
    -   **Read Operations:** Methods like `getIndex` employ a `StampedLock` in an optimistic read mode. This provides extremely low-overhead, non-blocking reads if no concurrent write is in progress. If a write collision is detected, it transparently falls back to a more traditional, but still highly efficient, shared read lock.
    -   **Write Operations:** Methods like `putAll` and `remove` acquire an exclusive write lock on the `StampedLock`. This guarantees data consistency by blocking all other readers and writers, but makes write operations a potential point of contention.
    -   **ID Generation:** A lock-free `AtomicInteger` is used for `nextIndex`, ensuring that unique index generation does not introduce an additional locking bottleneck during write operations.

    **Warning:** Due to the exclusive write lock, this component is optimized for "write-infrequently, read-many" workloads. Frequent writes from multiple threads will lead to significant lock contention and performance degradation.

## API Surface

The public API is intentionally minimal, exposing only the core functionality of index retrieval. State modification is handled internally by the asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIndex(K key) | int | O(1) | Retrieves the unique integer index for the given key. Returns Integer.MIN_VALUE if the key does not exist. |
| getIndexOrDefault(K key, int def) | int | O(1) | Retrieves the unique integer index for the given key, or returns the provided default value if the key does not exist. |
| getNextIndex() | int | O(1) | Returns the value of the next index that will be assigned, without incrementing it. |

## Integration Patterns

### Standard Usage

Developers typically do not interact with IndexedAssetMap directly. It is an internal implementation detail of the asset management framework. Higher-level services use it to resolve asset references into performant integer IDs.

```java
// Hypothetical usage by a higher-level AssetManager
// The manager holds a reference to an IndexedAssetMap instance.

public int getAssetId(String assetKey) {
    // The manager delegates the key-to-ID translation to the map.
    int id = this.internalAssetMap.getIndex(assetKey);

    if (id == Integer.MIN_VALUE) {
        throw new AssetNotFoundException("Asset not found: " + assetKey);
    }
    return id;
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new IndexedAssetMap()`. Its lifecycle is managed exclusively by the asset loading pipeline to ensure data integrity and proper registration.
-   **External Modification:** Do not attempt to modify the map's state from outside the asset loading system. The `putAll` and `remove` methods are protected for a reason; bypassing them will break consistency between the key-to-index map and the actual asset storage.
-   **Assuming Index Persistence:** Do not cache and store the integer indexes generated by this class across application sessions (e.g., saving them to a file). The indexes are generated at runtime and are not guaranteed to be the same between different runs of the game.

## Data Pipeline

The IndexedAssetMap is a critical transformation step in the asset data pipeline, converting symbolic names into numerical identifiers.

> Flow:
> Asset File on Disk -> Asset Loader Service -> `putAll` -> **IndexedAssetMap** (Key is mapped to a new integer ID) -> ID is propagated to engine systems (Rendering, ECS, etc.) for fast lookups.

