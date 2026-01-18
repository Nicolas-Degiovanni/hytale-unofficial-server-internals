---
description: Architectural reference for ItemToolSpec
---

# ItemToolSpec

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Model / Asset

## Definition
```java
// Signature
public class ItemToolSpec
   implements JsonAssetWithMap<String, DefaultAssetMap<String, ItemToolSpec>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ItemToolSpec> {
```

## Architecture & Concepts
The ItemToolSpec class is a data-centric model that defines the properties of a tool when used for a specific purpose, such as mining a particular type of block. It is not a service or manager, but rather a static data record loaded from configuration files.

This class is a fundamental component of the Hytale **Asset System**. Its primary role is to serve as the in-memory representation of a corresponding JSON asset file. The static `CODEC` field is the most critical architectural element; it declaratively defines the entire serialization and deserialization contract between the JSON on disk and the Java object in memory. This includes field mapping, data type conversion, validation rules, and post-processing logic.

ItemToolSpec acts as a bridge between three core domains:
1.  **Game Configuration:** Data defined by designers in JSON files.
2.  **Server Game Logic:** The running server uses these objects to determine tool effectiveness, sounds, and other behaviors.
3.  **Network Protocol:** The `NetworkSerializable` interface ensures this server-side configuration can be efficiently transmitted to the client via the `toPacket` method.

The class heavily utilizes the global `AssetRegistry` to store and retrieve all loaded instances, making tool specifications universally accessible throughout the server application.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the `AssetStore` during the server's asset loading phase at startup. The static `CODEC` is used to deserialize JSON configuration files into ItemToolSpec objects. **WARNING:** Manual instantiation is an anti-pattern and will result in a partially initialized, non-functional object.
-   **Scope:** Application-scoped. Once loaded, all ItemToolSpec instances are cached in a static, in-memory `ASSET_STORE` and persist for the entire lifetime of the server. They are treated as immutable data records after initial loading.
-   **Destruction:** Instances are destroyed and garbage collected only upon server shutdown when the `AssetRegistry` and its associated class loaders are released.

## Internal State & Concurrency
-   **State:** The object's state is considered **effectively immutable** after the `afterDecode` lifecycle hook runs the `processConfig` method. While the fields are not declared `final`, they are not designed to be mutated at runtime. The class contains a `SoftReference` to a cached network packet, which is a mutable cache field used for performance optimization.
-   **Thread Safety:** The class is **thread-safe for read operations**. Any game logic thread can safely access its properties (e.g., `getPower`, `getQuality`) without external synchronization. The `toPacket` method's internal caching mechanism is benign in a concurrent environment; in the worst-case scenario of a race condition, multiple identical packet objects might be created, but the final state remains consistent.

## API Surface
The public API is primarily for accessing pre-configured data and facilitating network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, static repository for all ItemToolSpec assets. |
| getAssetMap() | static DefaultAssetMap | O(1) | Provides direct map access to all loaded assets, keyed by their ID. |
| toPacket() | com.hypixel.hytale.protocol.ItemToolSpec | O(1) | Converts the server-side data model into a network-optimized DTO. Uses a soft-referenced cache. |
| getGatherType() | String | O(1) | Returns the unique identifier for this tool specification. |
| getPower() | float | O(1) | Returns the effectiveness or power level of the tool for this specification. |
| getHitSoundLayerIndex() | int | O(1) | Returns the pre-calculated integer index for the associated hit sound. |

## Integration Patterns

### Standard Usage
Developers should never create an ItemToolSpec directly. Instead, they should retrieve a pre-loaded instance from the global asset map using its unique identifier.

```java
// How a developer should normally use this
// Retrieve the map of all loaded tool specifications
DefaultAssetMap<String, ItemToolSpec> toolSpecs = ItemToolSpec.getAssetMap();

// Get a specific configuration, for example, for using an iron pickaxe on stone
ItemToolSpec spec = toolSpecs.get("iron_pickaxe_on_stone");

if (spec != null) {
    float power = spec.getPower();
    // ... use the spec for game logic calculations
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ItemToolSpec()`. This bypasses the asset loading pipeline, leaving critical fields like `hitSoundLayerIndex` uninitialized and preventing the object from being registered in the global store.
-   **Runtime Modification:** Do not attempt to modify the state of an ItemToolSpec object after it has been loaded. The system design relies on this data being a static, immutable record.
-   **Early Access:** Do not call `getAssetStore()` or `getAssetMap()` before the server's main asset loading phase is complete. This will result in a `NullPointerException` or an empty map.

## Data Pipeline
The ItemToolSpec class is a key stage in the asset configuration pipeline. Data flows from a human-readable file on disk to an optimized, in-memory object used by the server.

> Flow:
> JSON File (`*.json`) -> `AssetStore` Deserializer (using `ItemToolSpec.CODEC`) -> **ItemToolSpec Instance** -> `processConfig()` (Resolves Sound ID to Index) -> In-Memory `AssetMap` Cache -> Game Logic Access -> `toPacket()` -> Network Protocol Object

---

