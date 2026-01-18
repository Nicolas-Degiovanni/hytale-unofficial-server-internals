---
description: Architectural reference for ObjectiveLocationMarkerAsset
---

# ObjectiveLocationMarkerAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config
**Type:** Data Asset

## Definition
```java
// Signature
public class ObjectiveLocationMarkerAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ObjectiveLocationMarkerAsset>> {
```

## Architecture & Concepts
The ObjectiveLocationMarkerAsset class is a data contract that defines a specific, triggerable location related to an in-game objective. It is not a service or a manager; rather, it is a passive, data-centric object that represents configuration loaded from JSON files.

This class acts as the in-memory representation of an objective marker defined by game designers. Its primary architectural feature is the static **CODEC** field, an AssetBuilderCodec instance. This codec is the sole authority for deserializing the JSON configuration into a valid Java object. It declaratively defines the mapping between JSON keys (e.g., "Area", "TriggerConditions") and the object's fields, enforces data validation rules, and performs post-processing logic.

Once loaded by the engine's AssetStore, these objects serve as immutable configuration that higher-level systems, such as an objective or quest manager, can query to determine the rules and spatial boundaries for adventure mode progression.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale AssetStore during the engine's asset loading phase. The static CODEC is invoked by the store to deserialize a corresponding JSON file into a new ObjectiveLocationMarkerAsset object. Manual instantiation is strictly forbidden.
- **Scope:** Application-scoped. After initial loading, all instances are cached within a static, centrally managed AssetStore. They persist for the entire lifetime of the game session and are shared globally.
- **Destruction:** Instances are garbage collected only when the entire AssetRegistry is cleared, which typically occurs on application shutdown.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable after creation. All fields are populated once during the deserialization process defined by the CODEC. A critical post-processing step occurs in the `afterDecode` hook, which transforms the human-readable `environmentIds` array into a performance-optimized `environmentIndexes` integer array for fast runtime lookups. This derived state is also set only once.

- **Thread Safety:** The class is thread-safe for read operations. As its state is fixed after the initial loading phase, multiple threads can safely access its properties without external synchronization.

    **Warning:** The static `getAssetStore` method employs a lazy-initialization pattern. While generally safe due to the engine's single-threaded bootstrap sequence, calling it from multiple threads simultaneously before it has been initialized could lead to a race condition. All asset stores should be considered pre-warmed before multi-threaded game logic begins.

## API Surface
The public API is composed entirely of simple data accessors. There are no methods that mutate the object's state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique asset key for this marker. |
| getObjectiveTypeSetup() | ObjectiveTypeSetup | O(1) | Retrieves the core objective configuration. |
| getArea() | ObjectiveLocationMarkerArea | O(1) | Retrieves the spatial definition for this marker. |
| getEnvironmentIds() | String[] | O(1) | Returns the raw environment IDs where this marker is valid. |
| getEnvironmentIndexes() | int[] | O(1) | Returns the cached, sorted integer indexes for environments. |
| getTriggerConditions() | ObjectiveLocationTriggerCondition[] | O(1) | Returns the set of conditions for this marker. |

## Integration Patterns

### Standard Usage
Developers should never create instances of this class directly. The correct pattern is to retrieve a pre-loaded asset from the global asset map using its known identifier.

```java
// Retrieve a specific, pre-configured objective marker
DefaultAssetMap<String, ObjectiveLocationMarkerAsset> assetMap = ObjectiveLocationMarkerAsset.getAssetMap();
ObjectiveLocationMarkerAsset marker = assetMap.get("my_dungeon_entry_marker");

if (marker != null) {
    // Use the marker's configuration in game logic
    ObjectiveLocationMarkerArea area = marker.getArea();
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ObjectiveLocationMarkerAsset()`. The resulting object will be uninitialized, lack an ID, and fail in any system that expects a valid asset. All instances must be loaded via the AssetStore from a JSON source.
- **State Mutation:** Do not use reflection or other means to modify the fields of a retrieved asset. Downstream systems rely on this configuration being immutable and consistent throughout the game session.
- **Runtime Asset Loading:** While possible, querying the AssetStore for an asset for the first time during performance-critical game loop code can cause stuttering as the system may need to perform file IO and deserialization. Ensure all necessary objective assets are loaded during designated loading screens.

## Data Pipeline
The flow of data from configuration file to a usable in-memory object is managed entirely by the AssetStore framework.

> Flow:
> `objective_marker.json` (on disk) -> AssetStore Loader -> **AssetBuilderCodec** (Deserialization, Validation, Post-processing) -> **ObjectiveLocationMarkerAsset** (Cached in static AssetMap) -> Game Systems (Runtime lookup via `getAssetMap()`)

