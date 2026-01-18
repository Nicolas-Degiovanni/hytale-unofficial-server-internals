---
description: Architectural reference for LookupTableAssetMap
---

# LookupTableAssetMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Component / Data Structure

## Definition
```java
// Signature
public class LookupTableAssetMap<K, T extends JsonAssetWithMap<K, LookupTableAssetMap<K, T>>> extends AssetMapWithIndexes<K, T> {
```

## Architecture & Concepts
The LookupTableAssetMap is a high-performance, specialized data structure designed for O(1) asset retrieval using integer indices. It is not a general-purpose map. Its primary role within the engine's asset system is to serve as the final, optimized storage for assets that have a canonical, non-sparse integer ID, such as blocks, items, or other registered game objects.

This class acts as a bridge between the flexible, key-based world of asset files (e.g., a key of type ResourceLocation) and the performance-critical runtime world that requires immediate, index-based access (e.g., `getBlockByID(10)`).

It achieves this by maintaining an internal native array (`T[]`) which is directly indexed by the asset's integer ID. The logic for converting a symbolic key `K` into an integer index is not contained within this class. Instead, it is injected at construction time via functional interfaces:
*   **indexGetter:** A function that resolves a key `K` to its integer index.
*   **maxIndexGetter:** A function that supplies the current maximum possible index, used to govern the size of the internal array.

This dependency injection pattern makes LookupTableAssetMap a flexible component, configured by a higher-level registry that owns the canonical mapping of keys to integer IDs.

## Lifecycle & Ownership
- **Creation:** A LookupTableAssetMap is never instantiated directly by game logic. It is created and owned by a central asset management service, such as the AssetStore. During its construction, the AssetStore injects the critical `indexGetter` and `maxIndexGetter` functions, which are typically bound to a master registry (e.g., BlockRegistry).

- **Scope:** The object's lifetime is bound to an asset loading session. It persists as long as the assets it contains are considered active. A new instance may be created during major state changes, such as connecting to a server with a different set of registered assets.

- **Destruction:** The object is eligible for garbage collection when its owning asset service is reset or destroyed. The `clear` method will nullify its internal array and release references to all contained assets, but the object itself is managed by the Java garbage collector.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its core state is the internal `array` field, which is frequently re-allocated and modified as assets are loaded, unloaded, or reloaded. It is the canonical storage for index-based lookups, not a write-through cache.

- **Thread Safety:** The class is designed for concurrent access, but with a specific performance profile.
    - **Write Operations:** All methods that mutate the internal array (`putAll`, `remove`, `clear`) are fully synchronized using a `ReentrantLock`. This ensures that asset loading and unloading operations are atomic and prevents corruption of the internal array.
    - **Read Operations:** The primary read method, `getAsset(int)`, is **not** locked. It performs a direct, non-synchronized array access. This provides maximum read performance but relies on the atomicity of array reference reads in the JVM.

    **WARNING:** While `getAsset` is safe from exceptions, it is theoretically possible for a reader thread to observe a stale state during the brief moment a `resize` operation is swapping out the internal array reference. Code should not depend on the identity of the array itself remaining constant.

## API Surface
The public API is minimal, reflecting its role as an internal component. The primary methods are intended for use by the asset system, not end-users.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsset(int index) | T | O(1) | Retrieves an asset by its direct integer index. Returns null if the index is out of bounds or no asset exists at that index. This is the primary, high-performance read path. |
| putAll(...) | void | O(N) | **Internal API.** Populates the map with a batch of loaded assets. Triggers internal locking and a potential resize of the backing array. |
| remove(...) | Set<K> | O(M) | **Internal API.** Removes a set of assets by their keys. Triggers internal locking and a potential resize to shrink the backing array. |
| clear() | void | O(1) | **Internal API.** Removes all assets and resets the internal array to a zero-length state. |

## Integration Patterns

### Standard Usage
A developer should never interact with this class directly. Access is always mediated by a higher-level registry or manager that abstracts away the storage details.

```java
// The AssetStore or a similar service configures and holds the LookupTableAssetMap.
// Game code interacts with a public-facing registry.

// Example: Retrieving a block asset from a central registry
// This call is internally routed to LookupTableAssetMap.getAsset()
Block stone = BlockRegistry.getById(1);

if (stone != null) {
    // Use the fast-retrieved asset
    world.setBlock(position, stone);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new LookupTableAssetMap()`. The object is useless without the correct `indexGetter` and `maxIndexGetter` functions provided by the engine's central registries. Incorrectly providing these will lead to severe data corruption or runtime exceptions.

- **External Mutation:** Do not call `putAll`, `remove`, or `clear` from game logic. These methods are strictly part of the asset loading and hot-reloading pipeline and must be managed by the AssetStore to ensure engine stability.

- **Unsafe Iteration:** Do not attempt to get and iterate over the internal array via reflection. The array reference can be swapped at any time by another thread during a `resize` operation, leading to unpredictable behavior and `ArrayIndexOutOfBoundsException`.

## Data Pipeline
The LookupTableAssetMap is a terminal destination for assets in the loading pipeline and the starting point for runtime lookups.

> **Loading Flow:**
> Asset File (`.json`) -> AssetCodec (Deserialization) -> AssetStore (Orchestration) -> **LookupTableAssetMap.putAll** -> Internal `T[]` Array

> **Runtime Lookup Flow:**
> Game Code (`BlockRegistry.getById(1)`) -> **LookupTableAssetMap.getAsset(1)** -> Asset Instance `T`

