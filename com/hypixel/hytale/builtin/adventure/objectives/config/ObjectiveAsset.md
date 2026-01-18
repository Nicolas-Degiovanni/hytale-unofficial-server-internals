---
description: Architectural reference for ObjectiveAsset
---

# ObjectiveAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config
**Type:** Configuration Asset

## Definition
```java
// Signature
public class ObjectiveAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ObjectiveAsset>> {
```

## Architecture & Concepts

The ObjectiveAsset class is a data-driven configuration model that serves as a blueprint for in-game objectives. It is not an active game component but rather a static, deserialized representation of an objective's properties as defined in JSON asset files. Its primary role is to define the structure, tasks, and completion rewards of a single objective type.

The core of this class's architecture is the static **CODEC** field, an AssetBuilderCodec. This codec dictates the entire serialization contract, mapping JSON properties like *TaskSets* and *Completions* to the object's fields. It also handles sophisticated features like property inheritance from parent assets and post-deserialization logic via the *afterDecode* hook, which is used to generate default localization keys.

Instances of ObjectiveAsset are managed by the Hytale Asset System. Upon loading, they are registered in a dedicated, globally accessible **AssetStore**. Runtime systems do not instantiate this class directly; instead, they query this central store using a string identifier to retrieve the immutable objective definition.

The class also contains business logic for validation, such as the *isValidForPlayer* and *isValidForMarker* methods. These functions ensure that the tasks defined within an objective are compatible with the entity that will hold the objective (e.g., a player versus a static world marker), preventing invalid configurations at runtime.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale Asset System during the asset loading phase at game startup. The static **CODEC** is invoked to deserialize a corresponding JSON file into a new ObjectiveAsset object. Direct instantiation by developers is an anti-pattern and will result in an unmanaged, incomplete object.
-   **Scope:** An ObjectiveAsset instance, once loaded, persists for the entire application lifetime. It is stored within a static AssetStore and is treated as an immutable singleton definition for a specific objective ID.
-   **Destruction:** The object is garbage collected when the AssetStore is cleared, which typically occurs during game shutdown or a full asset reload initiated by development tools.

## Internal State & Concurrency

-   **State:** The internal state of an ObjectiveAsset is considered **immutable** after the asset loading and decoding process is complete. While its fields are not explicitly marked as final, the system design mandates that they are populated once by the CODEC and never modified at runtime.
-   **Thread Safety:** The object is inherently thread-safe for read operations, as its state does not change after initialization. Game systems can safely access its properties from any thread.

    **Warning:** The static `getAssetStore` method performs a lazy initialization of the backing ASSET_STORE. While the engine's startup sequence serializes asset loading, calling this method from a custom, unmanaged thread during early initialization could theoretically lead to a race condition. Access should be confined to post-initialization game logic.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, shared registry for all ObjectiveAsset definitions. |
| getAssetMap() | static DefaultAssetMap | O(1) | A convenience method to directly access the map of assets from the store. |
| isValidForPlayer() | boolean | O(N) | Validates if all contained tasks are compatible with a player entity. N is the total number of tasks. |
| isValidForMarker() | boolean | O(N) | Validates if all contained tasks are compatible with a world marker entity. N is the total number of tasks. |

## Integration Patterns

### Standard Usage

The correct pattern for using an ObjectiveAsset is to retrieve a pre-loaded definition from the global AssetStore using its unique string identifier. This definition is then used to configure runtime logic, such as creating an active objective for a player.

```java
// Retrieve the immutable objective definition from the central registry
ObjectiveAsset objectiveDefinition = ObjectiveAsset.getAssetStore().get("adventure:main_quest_01");

if (objectiveDefinition != null && objectiveDefinition.isValidForPlayer()) {
    // Use the definition to instantiate and configure a runtime objective state for a player
    player.getObjectiveState().beginObjective(objectiveDefinition);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ObjectiveAsset()`. This bypasses the asset loading pipeline, the CODEC deserialization, and the global registry. The resulting object will be malformed, lack critical data, and will not be recognized by other game systems.
-   **State Mutation:** Do not modify the fields of a retrieved ObjectiveAsset at runtime. These objects are shared definitions. Altering one will cause unpredictable and inconsistent behavior for all parts of the game that reference that objective.
-   **Assuming Existence:** Do not assume an asset exists. Always check for null after retrieving an asset from the store, as a missing or malformed JSON file will result in a failed load.

## Data Pipeline

The flow of data from a configuration file to a usable in-memory object is managed entirely by the Asset System.

> Flow:
> JSON File on Disk -> AssetStore Loader -> **ObjectiveAsset.CODEC** (Deserialization & Inheritance) -> **ObjectiveAsset Instance** -> Global AssetStore Registry -> Runtime System Query

