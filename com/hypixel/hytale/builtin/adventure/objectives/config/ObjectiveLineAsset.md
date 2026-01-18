---
description: Architectural reference for ObjectiveLineAsset
---

# ObjectiveLineAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config
**Type:** Data Asset

## Definition
```java
// Signature
public class ObjectiveLineAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ObjectiveLineAsset>> {
```

## Architecture & Concepts

The ObjectiveLineAsset class represents a structured, loadable configuration for a single stage or "line" within a larger quest or objective system. It is not a service or manager, but rather a data-centric object that defines a sequence of objectives and their relationships.

This class is a core component of Hytale's declarative, data-driven design. Instead of hard-coding quest logic, developers define quest flows in JSON files, which are then deserialized into ObjectiveLineAsset instances by the engine's asset pipeline.

The static **CODEC** field, an instance of AssetBuilderCodec, is the most critical architectural element. It dictates the entire serialization contract, including:
-   **Deserialization:** Mapping JSON keys like *Category* and *ObjectiveIds* to the corresponding class fields.
-   **Inheritance:** The ability for one objective line asset to inherit properties from a parent asset, reducing configuration duplication.
-   **Validation:** Applying rules such as ensuring *ObjectiveIds* is not empty, contains unique values, and that all referenced IDs exist in the ObjectiveAsset registry.
-   **Post-Processing:** The *afterDecode* hook automatically formats localization keys, ensuring consistency across all assets.

An ObjectiveLineAsset acts as a node in a directed graph of quest progression. It contains an ordered list of individual objectives (*objectiveIds*) that must be completed. Upon completion of all objectives in this line, the *nextObjectiveLineIds* field provides the keys to the subsequent ObjectiveLineAsset nodes, allowing for branching or linear quest chains.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale **AssetStore** framework during the engine's asset loading phase. The static CODEC field is invoked by the asset loader, which uses the provided `ObjectiveLineAsset::new` constructor reference to instantiate the object before populating its fields from the source JSON.
-   **Scope:** Once loaded, an ObjectiveLineAsset is cached in a static, centrally managed AssetStore. It is effectively a global, read-only singleton for its given asset ID and persists for the entire application lifetime.
-   **Destruction:** The object and its containing AssetStore are eligible for garbage collection only when the AssetRegistry is globally cleared, typically during client or server shutdown.

## Internal State & Concurrency

-   **State:** The state of an ObjectiveLineAsset is **effectively immutable** post-initialization. While its fields are not declared as final, they are populated only once during the asset decoding process. The game logic should treat these instances as read-only configuration data.
-   **Thread Safety:** This class is **not thread-safe** for mutation. However, the architectural pattern of the engine ensures safety in practice: assets are loaded and written to on a dedicated asset-loading thread, and subsequently read by game logic threads (e.g., the main game loop) without modification.

    **Warning:** The static `getAssetStore` method contains a lazy initialization block that is not synchronized. Calling this method from multiple threads simultaneously before the store is initialized can lead to a race condition. The engine's bootstrap process must ensure that the AssetRegistry is fully populated on a single thread before multi-threaded access is permitted.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, cached store for all ObjectiveLineAsset instances. |
| getAssetMap() | static DefaultAssetMap | O(1) | Convenience method to access the underlying map of asset IDs to instances from the store. |
| getNextObjectiveId(String) | String | O(N) | Performs a linear scan of the internal *objectiveIds* array to find the objective that follows the provided one. N is the number of objectives in this line. |

## Integration Patterns

### Standard Usage

Game systems should never create instances of this class directly. The primary interaction pattern is to retrieve a pre-loaded instance from the static AssetStore using its unique string identifier.

```java
// Retrieve the central store for all objective lines
AssetStore<String, ObjectiveLineAsset, ?> store = ObjectiveLineAsset.getAssetStore();

// Fetch a specific objective line by its ID (e.g., "main_quest_part_1")
ObjectiveLineAsset questStage = store.get("main_quest_part_1");

if (questStage != null) {
    // Use the asset to drive game logic, e.g., get the first objective
    String firstObjectiveId = questStage.getObjectiveIds()[0];
    
    // Get the localization key for the UI
    String titleKey = questStage.getObjectiveTitleKey();
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new ObjectiveLineAsset()`. An instance created this way will be uninitialized, invalid, and not registered with the game's asset system. It will not function correctly.
-   **State Mutation:** Do not modify the fields of a retrieved ObjectiveLineAsset instance. The asset system treats this data as read-only configuration. Modifying it at runtime can lead to unpredictable behavior and desynchronization.
-   **Concurrent Initialization:** Do not call `getAssetStore` from multiple threads during the application's initial loading phase. This can cause a race condition. All asset stores should be initialized by a single bootstrap thread.

## Data Pipeline

The ObjectiveLineAsset is a destination for configuration data. Its pipeline is concerned with loading and validation, not runtime data processing.

> Flow:
> JSON Asset on Disk -> AssetStore Loader -> **AssetBuilderCodec** (Deserializes, Validates, Inherits) -> **ObjectiveLineAsset Instance** -> Cached in static AssetStore -> Read-only access by Game Logic Systems

