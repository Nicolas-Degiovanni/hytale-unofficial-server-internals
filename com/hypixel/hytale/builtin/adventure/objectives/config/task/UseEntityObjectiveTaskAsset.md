---
description: Architectural reference for UseEntityObjectiveTaskAsset
---

# UseEntityObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Configuration Asset

## Definition
```java
// Signature
public class UseEntityObjectiveTaskAsset extends CountObjectiveTaskAsset {
```

## Architecture & Concepts
The UseEntityObjectiveTaskAsset is a data-driven configuration class, not a live game-world service. It serves as an immutable blueprint for a specific type of adventure mode objective: one that requires a player to interact with a designated entity a set number of times.

This class is a specialization of CountObjectiveTaskAsset, inheriting the core concept of a counter that must reach a target value. Its primary architectural role is to extend this concept with data specific to entity interaction, such as a target entity identifier, an animation to play upon use, and dialog to display.

The most critical architectural feature is the static **CODEC** field. This object defines the contract for serialization and deserialization, allowing the Hytale engine to reliably load this task configuration from external data files (e.g., JSON). The engine's Asset Management system uses this codec to translate on-disk data into a validated, in-memory Java object. This pattern decouples game logic from content creation, enabling designers to define complex objectives without writing code.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the `new` keyword in game logic. They are instantiated by the engine's serialization framework, which invokes the static `CODEC` during the asset loading process. The protected default constructor exists solely to support this framework.
-   **Scope:** An instance of this asset is typically loaded when its parent quest or zone becomes active. It is managed by the Asset Manager and is treated as a shared, immutable resource. Its lifetime persists as long as the asset is required by the active game state, often for the duration of a player's session within a specific zone or quest line.
-   **Destruction:** The object is eligible for garbage collection once the Asset Manager releases all references to it. This typically occurs when a player completes the associated quest or unloads the zone containing the objective.

## Internal State & Concurrency
-   **State:** This class is designed to be **immutable**. After being deserialized from a data file, its internal state (taskId, animationIdToPlay, etc.) is fixed and does not change for its entire lifetime. Data is exposed only through getter methods.
-   **Thread Safety:** Due to its immutable nature, this asset is inherently **thread-safe**. It can be safely read and shared across multiple threads—such as the main game loop, UI threads, or background logic threads—without the need for locks or other synchronization primitives.

## API Surface
The public API is minimal, consisting of accessors for its configured data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTaskScope() | ObjectiveTaskAsset.TaskScope | O(1) | Returns the scope hint, PLAYER_AND_MARKER, for the objective system. |
| getTaskId() | String | O(1) | Retrieves the unique identifier of the entity that must be used. |
| getAnimationIdToPlay() | String | O(1) | Retrieves the identifier for the animation to play on the entity upon use. |
| getDialogOptions() | DialogOptions | O(1) | Returns the nested configuration object for the interaction dialog. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly in procedural game logic. Its primary interaction is with the content pipeline and the objective system. A game designer authors a data file, and the engine loads it. The objective system then uses the resulting asset to initialize a live player objective.

A hypothetical system would retrieve the asset and pass it to the objective manager to begin tracking.

```java
// A system retrieves the pre-loaded asset and starts the objective
ObjectiveAsset objective = assetManager.get("my_quest.objective_1");
ObjectiveTaskAsset taskAsset = objective.getTask("use_the_lever");

if (taskAsset instanceof UseEntityObjectiveTaskAsset useEntityTask) {
    // The ObjectiveSystem now has the blueprint to track player interaction
    // with the entity identified by useEntityTask.getTaskId().
    objectiveSystem.startTask(player, useEntityTask);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new UseEntityObjectiveTaskAsset()`. The object must be created via the engine's asset loading pipeline to ensure it is correctly managed and cached. Direct instantiation bypasses this system and will lead to untracked, unmanaged state.
-   **State Mutation:** Do not attempt to modify the state of this object after it has been created (e.g., via reflection). The engine relies on its immutability for thread safety and predictable behavior. Modifying a shared asset instance will cause undefined behavior across all systems using it.

## Data Pipeline
The UseEntityObjectiveTaskAsset is a product of the engine's asset deserialization pipeline. It represents a transformation from raw data into a structured, type-safe memory representation.

> Flow:
> Adventure Mode JSON File -> Engine Asset Loader -> **BuilderCodec** -> **UseEntityObjectiveTaskAsset Instance** -> Objective System -> Live PlayerObjectiveState

---
# UseEntityObjectiveTaskAsset.DialogOptions

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Configuration Asset

## Definition
```java
// Signature
public static class UseEntityObjectiveTaskAsset.DialogOptions {
```

## Architecture & Concepts
DialogOptions is a nested, static inner class that serves as a dedicated data container for dialog-related configuration. It is not a standalone asset but is always owned by a parent UseEntityObjectiveTaskAsset. Its purpose is to group and encapsulate all parameters required to display a dialog box when a player interacts with the target entity.

Like its parent, it is defined by a static **CODEC**, making it a fully integrated component of the engine's serialization system. The fields `entityNameKey` and `dialogKey` are not raw strings but are localization keys. The UI system will use these keys to look up the appropriate translated text for the player's configured language.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the `CODEC` of its parent, UseEntityObjectiveTaskAsset, during the deserialization of the "Dialog" block in the source data file.
-   **Scope:** Its lifetime is strictly bound to its parent UseEntityObjectiveTaskAsset. It is created and destroyed at the same time as its parent.
-   **Destruction:** Garbage collected when its parent UseEntityObjectiveTaskAsset is collected.

## Internal State & Concurrency
-   **State:** **Immutable**. Its fields are set once during creation by the codec and never change.
-   **Thread Safety:** **Thread-safe**. As an immutable data holder, it can be safely accessed from any thread without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntityNameKey() | String | O(1) | Retrieves the localization key for the entity's display name. |
| getDialogKey() | String | O(1) | Retrieves the localization key for the main dialog text. |

