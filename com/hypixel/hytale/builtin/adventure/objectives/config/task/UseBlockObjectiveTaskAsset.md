---
description: Architectural reference for UseBlockObjectiveTaskAsset
---

# UseBlockObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Data Transfer Object / Asset

## Definition
```java
// Signature
public class UseBlockObjectiveTaskAsset extends CountObjectiveTaskAsset {
```

## Architecture & Concepts

The UseBlockObjectiveTaskAsset is a data-driven configuration class that defines a specific type of task within Hytale's adventure objective system. It represents a goal that requires a player to interact with or "use" a specific type of block a certain number of times.

This class acts as an immutable blueprint, not a live game state object. It is deserialized from game asset files (likely JSON) at runtime using its static **CODEC** field. This is a core principle of Hytale's engine: game logic and content are decoupled, allowing designers to create complex objectives without writing new Java code.

It extends CountObjectiveTaskAsset, inheriting the fundamental behavior of a task that must be completed a set number of times. Its unique specialization is the **blockTagOrItemIdField**, which specifies the target block. This field can reference either a single, specific block ID or a "block tag," which represents a group of related blocks (e.g., "hytale:woods" or "hytale:ores"). This provides significant flexibility for quest design.

The engine's ObjectiveSystem uses instances of this class to configure live progress trackers for players. When a player performs a "use block" action, the system checks if the targeted block matches the criteria defined in this asset.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the Hytale serialization engine via the static **CODEC** field. This occurs when the AssetManager loads adventure mode configuration files from the game's data directory. It is never created manually in game logic.
-   **Scope:** This object is a shared, immutable asset. It is loaded once and persists in memory for as long as its parent asset bundle (e.g., a specific world or adventure module) is active. Multiple systems can reference the same instance.
-   **Destruction:** The object is eligible for garbage collection when the AssetManager unloads the corresponding asset bundle, for example, when a player exits a world or the game is shut down.

## Internal State & Concurrency
-   **State:** Immutable. All fields, including the inherited count and the specific blockTagOrItemIdField, are set during deserialization and are not intended to be modified thereafter. This design guarantees that the asset's definition remains consistent throughout its lifecycle.
-   **Thread Safety:** Inherently thread-safe due to its immutability. The same UseBlockObjectiveTaskAsset instance can be safely read by the game logic thread, UI threads, and potentially background asset loading threads without requiring locks or synchronization.

## API Surface

The public API is minimal, primarily exposing the configured data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTaskScope() | ObjectiveTaskAsset.TaskScope | O(1) | Returns PLAYER_AND_MARKER, indicating the task is tracked per-player and may have associated map markers. |
| getBlockTagOrItemIdField() | BlockTagOrItemIdField | O(1) | Retrieves the core configuration field that defines the target block or block tag for the objective. |
| matchesAsset0(ObjectiveTaskAsset) | boolean | O(1) | Internal engine method to compare this asset's configuration against another. Used for asset validation and potential deduplication. |

## Integration Patterns

### Standard Usage

This class is not used directly by most game logic developers. Instead, its behavior is defined in external asset files. The ObjectiveSystem consumes these assets to manage player progress.

A designer would define the task in a data file like so (conceptual example):

```json
{
  "type": "hytale:use_block",
  "descriptionId": "objective.myquest.use_lever",
  "count": 5,
  "BlockTagOrItemId": "hytale:lever"
}
```

The engine then uses this data to create a UseBlockObjectiveTaskAsset instance, which the ObjectiveSystem references to track player actions.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new UseBlockObjectiveTaskAsset()`. The engine relies on the **CODEC** for proper initialization and integration with the asset system. Manual creation bypasses this and will lead to an unmanaged, non-functional object.
-   **State Mutation:** Do not attempt to modify the state of a loaded asset instance, for example, through reflection. The engine assumes these objects are immutable, and changing them at runtime will cause undefined and unpredictable behavior for all systems referencing the asset.

## Data Pipeline

The flow of data from configuration to gameplay execution follows a clear, data-driven path.

> Flow:
> Game Asset File (JSON) -> Hytale Codec System -> **UseBlockObjectiveTaskAsset instance** -> ObjectiveSystem (references the asset) -> Player "Use Block" Action -> Game Event Bus -> ObjectiveSystem Listener (compares action to asset criteria) -> Player Objective Progress Update

