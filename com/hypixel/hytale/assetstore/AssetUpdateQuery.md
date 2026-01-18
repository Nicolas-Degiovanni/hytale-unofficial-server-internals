---
description: Architectural reference for AssetUpdateQuery
---

# AssetUpdateQuery

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class AssetUpdateQuery {
```

## Architecture & Concepts
The AssetUpdateQuery is a configuration object, not an active service. Its primary role is to serve as a declarative parameter object for the AssetStore, specifying the scope and behavior of an asset update operation. This class embodies the Parameter Object pattern, decoupling the initiator of an asset update from the complex internal logic of the AssetStore.

By encapsulating flags such as which caches to rebuild (e.g., block textures, models, map geometry) and whether to perform asset comparison, it provides a clean, type-safe API for controlling potentially expensive reloading and processing tasks. This design prevents the proliferation of boolean parameters in AssetStore methods and allows for fine-grained control over performance-critical operations.

The class is composed of two main parts:
1.  **AssetUpdateQuery:** The top-level container, holding general flags like *disableAssetCompare*.
2.  **RebuildCache:** A nested, immutable data structure that specifies exactly which categories of assets require their caches to be invalidated and rebuilt.

This structure allows systems to request asset updates with high precision, preventing unnecessary work. For example, a change to a single item's icon should not trigger a full rebuild of all world geometry.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand by any system requiring an asset update. Common configurations are provided as static singletons (DEFAULT, DEFAULT_NO_REBUILD) for convenience. Custom queries are constructed via the nested RebuildCacheBuilder. The caller is the owner of the created instance.
-   **Scope:** The object's lifetime is transient and typically confined to the scope of a single method call. It is created, passed as an argument to an AssetStore method, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java garbage collector. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** The AssetUpdateQuery and its nested RebuildCache class are **deeply immutable**. All fields are declared final and are set only during construction. This guarantees that a query's configuration cannot be altered after creation, ensuring predictable behavior within the asset system. The associated RebuildCacheBuilder is mutable, as is standard for the Builder pattern, but its purpose is to produce a final, immutable RebuildCache object.
-   **Thread Safety:** The class is inherently **thread-safe** due to its immutability. Instances can be safely created on one thread, passed to another (e.g., a background asset loading thread), and read without any risk of data races or a need for external synchronization. This is a critical feature for a high-performance, multi-threaded engine.

## API Surface
The primary API interaction is through the static instances or the builder pattern for the nested RebuildCache.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DEFAULT | AssetUpdateQuery | O(1) | A static instance configured to rebuild all asset caches. Use with caution. |
| DEFAULT_NO_REBUILD | AssetUpdateQuery | O(1) | A static instance configured to rebuild no asset caches. |
| RebuildCache.builder() | RebuildCacheBuilder | O(1) | Returns a new builder for creating a custom RebuildCache configuration. |
| RebuildCacheBuilder.build() | RebuildCache | O(1) | Constructs a final, immutable RebuildCache instance from the builder's state. |

## Integration Patterns

### Standard Usage
For custom or partial asset updates, the builder pattern is the required approach. This allows for precise control over which caches are affected.

```java
// Example: Rebuilding only item icons and model textures after a mod update.
AssetUpdateQuery.RebuildCache cachePolicy = AssetUpdateQuery.RebuildCache.builder()
    .setItemIcons(true)
    .setModelTextures(true)
    .build();

AssetUpdateQuery query = new AssetUpdateQuery(cachePolicy);

// The query is then passed to the central AssetStore
assetStore.updateAssets(query);
```

### Anti-Patterns (Do NOT do this)
-   **Indiscriminate Full Rebuilds:** Do not use AssetUpdateQuery.DEFAULT as a catch-all solution. Triggering a full rebuild of all caches is an extremely expensive operation that can cause significant frame drops or loading stalls. Always define the narrowest possible scope for an update.
-   **Builder Reuse:** The RebuildCacheBuilder is a stateful, one-shot object. Do not hold a reference to a builder and reuse it after calling the build method. This can lead to unpredictable query configurations. Always create a new builder for each distinct query.

## Data Pipeline
The AssetUpdateQuery does not process data itself; it is a command object that directs the flow of a larger data pipeline within the AssetStore.

> Flow:
> External Trigger (e.g., Mod Load, Resource Pack Change) -> System creates **AssetUpdateQuery** -> AssetStore.updateAssets(query) -> AssetStore reads query flags -> Targeted Cache Invalidation (e.g., ModelCache, TextureCache) -> Asset Reload & Reprocessing -> GPU Resource Upload

