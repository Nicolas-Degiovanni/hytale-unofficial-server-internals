---
description: Architectural reference for PortalType
---

# PortalType

**Package:** com.hypixel.hytale.server.core.asset.type.portalworld
**Type:** Data Asset

## Definition
```java
// Signature
public class PortalType implements JsonAssetWithMap<String, DefaultAssetMap<String, PortalType>> {
```

## Architecture & Concepts

The PortalType class is a data-driven configuration object, not a service or manager. It serves as an in-memory representation of a portal's properties, deserialized from a corresponding JSON asset file. This class is a fundamental component of the Hytale Asset System, which favors declarative data (JSON) over hard-coded logic.

Architecturally, PortalType acts as a data contract between the game's content files and the server's runtime logic. It defines the link between a physical portal entity in the world and the server instance it connects to, including specific gameplay rules, spawn behaviors, and item restrictions for that destination.

The core mechanism for its operation is the static `CODEC` field, an instance of AssetBuilderCodec. This codec provides a declarative schema that maps JSON keys directly to the Java fields of a PortalType instance. The entire collection of loaded portals is managed by a static, lazily-initialized AssetStore, which functions as a global registry. This allows any server system to query for a specific portal's configuration by its unique string identifier without needing a direct reference.

## Lifecycle & Ownership

-   **Creation:** PortalType instances are never created manually with the `new` keyword. They are exclusively instantiated by the Hytale AssetStore during the server's bootstrap or asset-loading phase. The `AssetBuilderCodec` orchestrates the construction and population of the object from a JSON source file.
-   **Scope:** Application-scoped. Once an asset is loaded into the static `ASSET_STORE`, it persists for the entire lifetime of the server process. Each unique portal ID corresponds to a single, shared instance.
-   **Destruction:** Instances are garbage collected only when the server shuts down or when the asset system performs a full reload, at which point the static `ASSET_STORE` reference is cleared and rebuilt.

## Internal State & Concurrency

-   **State:** Instances are effectively immutable after creation. All fields are populated once during deserialization by the codec, and there are no public setters to modify the state at runtime. This design ensures that configuration data remains consistent and predictable throughout the server session.
-   **Thread Safety:** Individual PortalType instances are thread-safe for read operations due to their immutable nature.

    **WARNING:** The static `getAssetStore` method contains a non-thread-safe lazy initialization pattern (`if (ASSET_STORE == null)`). If multiple threads attempt to access the asset store for the first time concurrently, it can lead to a race condition, potentially creating multiple AssetStore instances or resulting in undefined behavior. All initial access to this and other asset stores should be synchronized or confined to a single thread during server initialization.

## API Surface

The primary interaction with this class is through its static methods, which provide access to the central asset registry.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | Returns the central store for all PortalType assets. **Warning:** Not thread-safe on first call. |
| getAssetMap() | DefaultAssetMap | O(1) | A convenience method to get the underlying map of all loaded PortalType assets, keyed by ID. |
| getId() | String | O(1) | Returns the unique identifier for this portal type, matching the asset key. |
| getInstanceId() | String | O(1) | Returns the ID of the server instance this portal leads to. |
| getGameplayConfig() | GameplayConfig | O(1) | Retrieves the associated GameplayConfig asset by looking up its ID. This demonstrates inter-asset referencing. |

## Integration Patterns

### Standard Usage

To use a PortalType, you must retrieve it from the global asset map using its unique identifier. Never attempt to instantiate it directly.

```java
// Correctly retrieve a pre-loaded PortalType configuration
DefaultAssetMap<String, PortalType> portalMap = PortalType.getAssetMap();
PortalType zone1Portal = portalMap.getAsset("adventure_zone_1_entry");

if (zone1Portal != null) {
    String instance = zone1Portal.getInstanceId();
    // Logic to connect player to the specified instance...
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new PortalType()`. This bypasses the asset loading system and results in an uninitialized object with null fields, which will cause NullPointerExceptions at runtime. The object's state is entirely dependent on the codec-driven deserialization process.
-   **Concurrent Initialization:** Do not call `getAssetStore()` or `getAssetMap()` from multiple threads simultaneously during server startup. This can corrupt the static AssetStore singleton. Ensure all asset stores are initialized from a single, controlled thread.
-   **State Modification:** Do not use reflection to modify the internal state of a loaded PortalType instance. The system relies on these objects being immutable.

## Data Pipeline

The flow of data from a content file to a usable runtime object is managed entirely by the Asset System.

> Flow:
> `portal_type.json` on Disk -> Server Asset Loader -> **AssetBuilderCodec** (Deserializes JSON to Object) -> **PortalType Instance** -> Stored in static `ASSET_STORE` -> Game Logic via `PortalType.getAssetMap()`

