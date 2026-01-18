---
description: Architectural reference for ObjectiveTaskAsset
---

# ObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Data Model / Asset

## Definition
```java
// Signature
public abstract class ObjectiveTaskAsset {
```

## Architecture & Concepts

The ObjectiveTaskAsset class is an abstract base for all objective task configurations within Hytale's Adventure Mode system. It is not an active game component but rather a static data blueprint, deserialized from asset files (e.g., JSON) that define the structure and conditions of a quest or objective.

Its primary role is to serve as a polymorphic data container. The static `CODEC` field, a `CodecMapCodec`, is the central architectural feature. This codec enables the engine to read a "Type" identifier from an asset file and instantiate the corresponding concrete subclass (e.g., a `KillMobTaskAsset` or `GatherItemTaskAsset`). This pattern allows game designers to define new and varied task types purely through data, without requiring engine code changes.

This class encapsulates the universal properties shared by all tasks:
*   A localizable description identifier.
*   An array of `TaskConditionAsset` instances, which define the specific requirements for completion.
*   An array of `Vector3i` coordinates for displaying associated markers on the in-game map.

Subclasses are responsible for adding fields specific to their task type and implementing the `getTaskScope` method to define where and how the task can be completed.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale Codec serialization system during the asset loading phase. The top-level `CodecMapCodec` reads an objective configuration file, identifies the task "Type", and invokes the appropriate builder to construct a concrete `ObjectiveTaskAsset` instance. Direct instantiation is an anti-pattern.
-   **Scope:** An `ObjectiveTaskAsset` is an immutable data object whose lifetime is bound to its parent objective configuration. It persists in memory as long as the objective is active or cached by the server's objective management system.
-   **Destruction:** The object is eligible for garbage collection when its parent objective configuration is unloaded. This typically occurs when a world is shut down or a game mode deinitializes. There are no explicit destruction methods.

## Internal State & Concurrency

-   **State:** The object is designed to be effectively immutable after creation. Its core fields (`descriptionId`, `taskConditions`, `mapMarkers`) are populated once by the codec and should not be modified at runtime. The private `defaultDescriptionId` field is a lazily-initialized cache, but its mutation is idempotent and does not affect the external contract of the class.

-   **Thread Safety:** The class is conditionally thread-safe. It is safe for concurrent reads, which is its intended use case after the initial asset loading phase. **WARNING:** The class is not safe for concurrent writes. Modifying the contents of the arrays returned by `getTaskConditions` or `getMapMarkers` from any thread is an unsupported operation and will lead to unpredictable behavior and data corruption across the system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDescriptionKey(objectiveId, taskSetIndex, taskIndex) | String | O(1) | Returns the explicit description ID if present, otherwise dynamically constructs and caches a default localization key. |
| getTaskConditions() | TaskConditionAsset[] | O(1) | Returns the array of conditions required to complete this task. |
| getMapMarkers() | Vector3i[] | O(1) | Returns the array of world coordinates for associated map markers. |
| getTaskScope() | TaskScope | O(1) | Abstract method. Returns an enum defining the context in which the task can be completed (e.g., by a player, at a marker). |
| matchesAsset(task) | boolean | O(N) | Performs a deep comparison against another task asset. Complexity is proportional to the number of conditions and markers. |

## Integration Patterns

### Standard Usage

Developers should never create instances of this class directly. Instead, they are retrieved from a higher-level, fully-loaded objective configuration asset. The primary interaction is reading its data to drive game logic.

```java
// Assume 'objectiveConfig' is an asset loaded by the engine
ObjectiveTaskAsset task = objectiveConfig.getTaskSet(0).getTask(0);

// Use the key to get a localized string for the UI
String descriptionKey = task.getDescriptionKey("q_main_story", 0, 0);
String localizedText = I18n.format(descriptionKey);

// Game logic iterates over conditions to check for task completion
for (TaskConditionAsset condition : task.getTaskConditions()) {
    if (!condition.isMet(player)) {
        // Task is not yet complete
        return;
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new` on a concrete subclass of `ObjectiveTaskAsset`. All tasks must be defined in data files and loaded via the engine's asset pipeline to ensure they are correctly registered and managed.
-   **State Mutation:** Do not modify the arrays returned by `getTaskConditions` or `getMapMarkers`. These are considered immutable after loading. Modifying them at runtime constitutes a violation of the class contract.
-   **Ignoring Scope:** Game logic that checks for task completion must respect the value of `getTaskScope`. For example, attempting to complete a `MARKER` scoped task when the player is not near the designated location is incorrect.

## Data Pipeline

The `ObjectiveTaskAsset` is a product of the engine's asset deserialization pipeline. It represents structured data that has been loaded into memory for consumption by the game simulation.

> Flow:
> Objective Configuration File (.json) -> Hytale Asset Loader -> `CodecMapCodec` Deserializer -> **ObjectiveTaskAsset Instance** -> Objective Management Service -> Game Logic & UI Systems

