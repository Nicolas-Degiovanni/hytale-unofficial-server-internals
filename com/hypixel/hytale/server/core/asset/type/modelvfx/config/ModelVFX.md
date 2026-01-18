---
description: Architectural reference for ModelVFX
---

# ModelVFX

**Package:** com.hypixel.hytale.server.core.asset.type.modelvfx.config
**Type:** Data Model / Asset Type

## Definition
```java
// Signature
public class ModelVFX
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ModelVFX>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ModelVFX> {
```

## Architecture & Concepts

The ModelVFX class is a server-side data model that represents a single, configurable visual effect asset. It functions as a strongly-typed container for properties defined in external configuration files, likely JSON, which describe how visual effects like highlights, dissolves, or glows should behave on 3D models.

This class is a fundamental component of the Hytale **Asset System**. It does not contain any logic for rendering or effect management; its sole responsibility is to hold validated configuration data.

The architectural cornerstone of this class is the static **CODEC** field, an instance of AssetBuilderCodec. This codec defines the schema for a ModelVFX asset, mapping JSON properties to Java fields and enforcing validation rules (e.g., non-null fields, value ranges) during the asset loading process. This ensures that any ModelVFX object in memory is guaranteed to be valid and structurally correct.

Integration with the wider engine is managed through a static **AssetStore**. The static getAssetStore method provides access to a globally managed, lazily initialized registry that contains all loaded ModelVFX assets. This store uses an IndexedLookupTableAssetMap, which provides highly efficient O(1) lookup of assets by their string identifier.

Finally, its implementation of the NetworkSerializable interface marks it as a bridge between the server's game state and the client. The toPacket method serializes the configuration into a distinct protocol-level DTO, preparing it for transmission over the network.

## Lifecycle & Ownership

-   **Creation:** Instances of ModelVFX are not intended to be manually instantiated by developers. They are created exclusively by the Hytale Asset System during server initialization or resource reloading. The system uses the static CODEC to deserialize asset files from disk into new ModelVFX objects. The default constructor is the designated entry point for this deserialization process.

-   **Scope:** The lifetime of a ModelVFX instance is coupled to the global AssetStore. Once loaded, an instance persists in memory for the entire server session. These objects should be treated as read-only, shared resources, effectively acting as singletons for their unique asset ID.

-   **Destruction:** Instances are dereferenced and marked for garbage collection only when the AssetStore is cleared, which typically occurs during a full server shutdown or a manual asset reload command.

## Internal State & Concurrency

-   **State:** A ModelVFX object is **effectively immutable** after its creation. Its state is fully populated by the CODEC during the asset loading phase. While its fields are not explicitly marked as final, the public API exposes no methods for mutation. It is designed to be a read-only configuration template.

-   **Thread Safety:** The class is **thread-safe for read operations**. Because its internal state does not change after initialization, multiple threads can safely access the same ModelVFX instance to retrieve configuration data without requiring locks or synchronization.

    **Warning:** The static getAssetStore method contains a lazy initializer. While the surrounding AssetRegistry is likely initialized in a single-threaded context at startup, calling this method from multiple threads before it has been initialized for the first time could theoretically lead to a race condition. Post-initialization, access is safe.

## API Surface

The public API is primarily for retrieving configuration data or accessing the central asset store. Simple getters like getLoopOption or getCurveType are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, shared store for all ModelVFX assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Returns the underlying map from the asset store for direct access. |
| toPacket() | com.hypixel.hytale.protocol.ModelVFX | O(1) | Creates and returns a network-optimized packet from the asset's data. |
| getId() | String | O(1) | Returns the unique identifier string for this asset. |

## Integration Patterns

### Standard Usage

A game system, such as an EffectController, should retrieve a pre-loaded ModelVFX configuration from the central store using its unique ID. This object is then used as a template, often by converting it to a network packet to be sent to the client.

```java
// How a developer should normally use this
// Retrieve the central store for all ModelVFX assets
AssetStore<String, ModelVFX, ?> store = ModelVFX.getAssetStore();

// Get a specific VFX configuration by its ID
ModelVFX dissolveEffect = store.get("hytale:vfx_dissolve_fast");

if (dissolveEffect != null) {
    // Convert the configuration to a network packet
    com.hypixel.hytale.protocol.ModelVFX packet = dissolveEffect.toPacket();

    // Send the packet to the player's client
    player.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ModelVFX()`. Manually created instances will not be registered in the global AssetStore and will be missing critical data populated by the asset loader. This bypasses the entire asset management system.

-   **State Mutation:** Do not attempt to modify the state of a retrieved ModelVFX instance, for example, through reflection. These objects are shared across the entire server. Modifying one will cause unpredictable and inconsistent behavior for all systems that use that asset.

## Data Pipeline

The ModelVFX class is a key stage in the pipeline that transforms a configuration file on disk into a visual effect rendered on a player's screen.

> Flow:
> JSON Asset File -> AssetLoader -> **ModelVFX.CODEC** -> **ModelVFX Instance** -> AssetStore -> Game System -> toPacket() -> Network Layer -> Client Renderer

