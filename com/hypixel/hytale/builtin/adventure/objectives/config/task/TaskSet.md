---
description: Architectural reference for TaskSet
---

# TaskSet

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Configuration Model

## Definition
```java
// Signature
public class TaskSet {
```

## Architecture & Concepts

The TaskSet class is a data model that represents a structured group of tasks within a larger adventure mode objective. It is not a service or a manager; its sole purpose is to act as a strongly-typed container for configuration data loaded from game assets.

The most critical architectural component of this class is the static **CODEC** field. This `BuilderCodec` instance defines the contract between the Java object and its serialized representation, typically a JSON file. The Hytale asset loading pipeline uses this codec to deserialize a block of configuration into a valid TaskSet object at runtime.

This class effectively serves as a schema for a portion of an objective's definition. It ensures that any configured set of tasks adheres to a specific structure, containing a description identifier and a non-empty array of ObjectiveTaskAsset definitions. Its `getDescriptionKey` method provides a fallback mechanism, integrating it with the localization system to generate predictable, stable translation keys if a custom one is not provided in the asset file.

## Lifecycle & Ownership

-   **Creation:** TaskSet instances are created exclusively by the Hytale asset loading framework during the deserialization of objective configuration files. The static `CODEC` is invoked by a higher-level parser to construct the object. **WARNING:** Manual instantiation of this class during gameplay is an architectural violation.
-   **Scope:** The lifetime of a TaskSet instance is strictly bound to its parent configuration object, such as an ObjectiveAsset. It persists in memory as long as the parent objective definition is cached by the game engine.
-   **Destruction:** The object is marked for garbage collection when its parent ObjectiveAsset is unloaded or all references to it are released. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency

-   **State:** A TaskSet is **effectively immutable** post-initialization. While its fields are not declared final, the design pattern dictates that it is a read-only representation of static configuration data. Its state is not intended to be mutated after being loaded from an asset.
-   **Thread Safety:** The class is **conditionally thread-safe**. As a plain data holder, it is safe for concurrent reads from multiple threads *after* it has been fully constructed by the asset pipeline. It is not safe for mutation, and any external attempts to modify its internal arrays would be inherently unsafe without external locking, which is not a supported pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDescriptionId() | String | O(1) | Returns the custom description identifier from the asset, or null if not present. |
| getDescriptionKey(objectiveId, taskSetIndex) | String | O(1) | Returns the definitive localization key. Uses the custom ID if available, otherwise generates a default key based on the objective's ID and this set's index. |
| getTasks() | ObjectiveTaskAsset[] | O(1) | Returns the array of tasks defined for this set. **WARNING:** The returned array should be treated as read-only. |

## Integration Patterns

### Standard Usage

A developer should never create a TaskSet directly. Instead, it is accessed from a parent configuration object that has been loaded by the engine.

```java
// Assume 'objectiveAsset' is a fully loaded configuration object
ObjectiveConfig config = objectiveAsset.getConfig();

// Iterate through the task sets defined in the asset
for (TaskSet taskSet : config.getTaskSets()) {
    ObjectiveTaskAsset[] tasks = taskSet.getTasks();
    
    // Game logic can now safely read the configured tasks
    processTasks(tasks);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new TaskSet()`. The object's state is meaningless without being populated by the asset pipeline via its `CODEC`.
-   **Runtime State Mutation:** Do not modify the `tasks` array or other fields after the object has been loaded. This breaks the contract that the object represents static game data and can lead to desynchronization and undefined behavior.

## Data Pipeline

The TaskSet class is a destination point in the asset loading data pipeline. It represents the transformation of raw configuration data into a usable in-memory object.

> Flow:
> Game Asset File (.json) -> Asset Loading Service -> **TaskSet.CODEC** (Deserialization & Validation) -> In-Memory **TaskSet** Object -> Objective Management System

