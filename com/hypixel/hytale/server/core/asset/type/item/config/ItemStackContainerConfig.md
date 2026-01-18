---
description: Architectural reference for ItemStackContainerConfig
---

# ItemStackContainerConfig

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient

## Definition
```java
// Signature
public class ItemStackContainerConfig {
```

## Architecture & Concepts
ItemStackContainerConfig is a passive data structure that defines the behavioral properties of an inventory or any game entity capable of holding items. It is not an active service but rather a configuration object, deserialized from game asset files during the engine's loading phase.

Its primary architectural role is to serve as a data-driven blueprint for container behavior. The class is designed to be instantiated and configured exclusively by the asset serialization system, using the static CODEC field. This approach decouples container logic from hard-coded values, allowing designers to tweak parameters like capacity and filtering rules directly in data files.

A critical design feature is the post-deserialization logic embedded within its CODEC. The `afterDecode` hook integrates directly with the AssetRegistry to convert a human-readable string `tag` into a performance-optimized integer `tagIndex`. This pre-computation step is essential for fast, runtime tag lookups, avoiding costly string comparisons in performance-sensitive game loops.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the asset loading pipeline via the static `ItemStackContainerConfig.CODEC`. Manual instantiation using the `new` keyword is strongly discouraged and considered an anti-pattern. A static `DEFAULT` instance exists for fallback scenarios where a configuration is not explicitly provided.
- **Scope:** The lifetime of an ItemStackContainerConfig object is bound to the asset that defines it, such as an item or a block entity definition. It is effectively a value object whose scope does not extend beyond its parent asset.
- **Destruction:** Managed by the Java Garbage Collector. When the parent asset is unloaded or dereferenced, the associated config object is marked for collection. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The object is mutable during the asset deserialization process. Once the `afterDecode` hook completes and the object is attached to its parent asset, it should be treated as **effectively immutable**. Modifying its state at runtime is unsupported and will lead to undefined behavior.
- **Thread Safety:** This class is **not thread-safe**. Direct mutation from multiple threads will corrupt its state. However, the `tagIndex` field is marked as `volatile`. This is a deliberate design choice to ensure that the value of `tagIndex`, which is written once on the asset loading thread, is safely and correctly published to all other threads (e.g., game logic threads) that may read it. This guarantees visibility without the overhead of a full lock.

**WARNING:** Do not modify fields after the asset loading phase. All reads of `tagIndex` from any thread are safe, but writes are not.

## API Surface
The public API is minimal, consisting only of getters for its configured properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCapacity() | short | O(1) | Returns the maximum number of items the container can hold. |
| getGlobalFilter() | FilterType | O(1) | Returns the filter rule that governs which items are allowed. |
| getTagIndex() | int | O(1) | Returns the pre-computed integer index for the item tag. Use this for all performance-critical tag checks. |

## Integration Patterns

### Standard Usage
This class is not used directly in procedural code. Instead, it is defined declaratively within an asset file (e.g., a JSON file for an item). The game engine's asset loader uses the `CODEC` to create and populate the object.

A developer's primary interaction is through the asset data, not Java code.

*Example Asset Definition (conceptual JSON):*
```json
{
  "name": "hytale:magic_pouch",
  "containerConfig": {
    "Capacity": 16,
    "GlobalFilter": "ALLOW_GEMS",
    "ItemTag": "magic_container"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new ItemStackContainerConfig()`. The internal `tagIndex` will not be initialized, leading to `Integer.MIN_VALUE` being returned and causing subtle bugs in inventory logic.
- **Runtime Modification:** Do not call setters (if they existed) or modify fields on a config object after it has been loaded. This breaks the "configuration as code" paradigm and can cause desynchronization if the asset is shared.
- **Using String Tag for Lookups:** Avoid accessing the internal `tag` field for comparisons in game logic. The `getTagIndex()` method provides a highly optimized integer that should be used for all tag-based filtering and lookups.

## Data Pipeline
The flow of data from asset file to a usable in-memory object is managed entirely by the asset system. The `CODEC` is the central component in this transformation.

> Flow:
> Game Asset File (JSON) -> Asset Deserializer -> **ItemStackContainerConfig.CODEC** -> `afterDecode` Hook (Calls AssetRegistry) -> Populated `ItemStackContainerConfig` Instance -> Attached to Parent Asset

