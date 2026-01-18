---
description: Architectural reference for BuilderToolItemReferenceAsset
---

# BuilderToolItemReferenceAsset

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Model / Asset

## Definition
```java
// Signature
public class BuilderToolItemReferenceAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, BuilderToolItemReferenceAsset>> {
```

## Architecture & Concepts
The BuilderToolItemReferenceAsset is a passive data structure that represents a manifest of related item assets. It is a fundamental component of the Hytale Asset System, acting as a typed, in-memory representation of a configuration file that groups multiple item identifiers under a single logical name.

Its primary architectural purpose is to decouple game logic from raw data files. Instead of parsing JSON manually, systems can query the AssetStore for a strongly-typed BuilderToolItemReferenceAsset object. For example, a "Wood Blocks" asset of this type would contain an array of strings referencing all individual wood block items, such as "hytale:oak_plank" and "hytale:spruce_log".

The most critical architectural element is the static **CODEC** field. This field declaratively defines the serialization contract between the raw JSON data on disk and this Java class. The engine's AssetStore uses this codec to automatically instantiate and populate these objects during the asset loading pipeline, ensuring data integrity and consistency. This class does not contain any logic; it is purely a data container.

## Lifecycle & Ownership
-   **Creation:** Instances are materialized exclusively by the Hytale **AssetStore** during the engine's asset loading phase. The static CODEC definition is consumed by the loading system to deserialize a corresponding JSON file into a new BuilderToolItemReferenceAsset object. Direct instantiation by developers is strictly forbidden.

-   **Scope:** The lifecycle of an instance is bound to the global AssetRegistry. Once loaded, an asset persists for the entire server or client session, residing within the static AssetStore cache. It is effectively a session-scoped, read-only object.

-   **Destruction:** Instances are marked for garbage collection when the AssetRegistry is cleared. This typically occurs during a server shutdown, when a client disconnects, or during a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** The internal state of a BuilderToolItemReferenceAsset instance is **effectively immutable** post-creation. Its fields, such as itemIds, are populated once by the CODEC during deserialization and are not designed to be modified at runtime. The class serves as a read-only data record.

-   **Thread Safety:** Instances are inherently thread-safe for read operations due to their immutable nature. Multiple threads can safely call getItems() or getId() on a shared instance without synchronization.

    **WARNING:** The static `getAssetStore()` method employs a lazy-initialization pattern. While the underlying AssetRegistry is expected to be thread-safe, concurrent calls to `getAssetStore()` from multiple threads *before the store has been initialized* could theoretically lead to a race condition. All access should occur on the main thread during system initialization or after ensuring the AssetRegistry has been fully populated.

## API Surface
The public API is designed for static, global access to the collection of assets and read-only access to instance data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static | O(1) | Retrieves the global map of all loaded assets of this type. Keyed by asset ID. |
| getAssetStore() | static | O(1) | Retrieves the backing AssetStore that manages the lifecycle of these assets. |
| getItems() | String[] | O(1) | Returns the array of item asset identifiers defined in this manifest. |
| getId() | String | O(1) | Returns the unique identifier for this specific asset manifest. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve the global asset map and query it for a specific manifest by its known identifier.

```java
// Retrieve the map containing all builder tool reference assets
DefaultAssetMap<String, BuilderToolItemReferenceAsset> allBuilderTools = BuilderToolItemReferenceAsset.getAssetMap();

// Get a specific reference asset by its unique ID
BuilderToolItemReferenceAsset woodTools = allBuilderTools.get("hytale:wood_building_blocks");

if (woodTools != null) {
    // Access the list of items for use in game logic (e.g., populating a creative menu)
    String[] itemIds = woodTools.getItems();
    for (String itemId : itemIds) {
        // ... process item
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BuilderToolItemReferenceAsset()`. The object will be uninitialized and detached from the asset management system, leading to unpredictable behavior and NullPointerExceptions.

-   **State Mutation:** Do not attempt to modify the array returned by `getItems()`. This array should be treated as read-only. Modifying it violates the asset system's immutability contract and can cause inconsistent state across the application.

-   **Premature Access:** Do not call `getAssetMap()` or `getAssetStore()` during early engine bootstrap before the AssetRegistry has completed its loading phase. This will result in an empty or incomplete asset map.

## Data Pipeline
The BuilderToolItemReferenceAsset is the result of a data transformation pipeline that begins with a raw file on disk and ends with a queryable, in-memory object.

> Flow:
> JSON File (`assets/.../buildertool/wood.json`) -> Engine Asset Loader -> **BuilderToolItemReferenceAsset.CODEC** -> **BuilderToolItemReferenceAsset Instance** -> Caching in AssetStore -> Game Logic (e.g., UI System)

