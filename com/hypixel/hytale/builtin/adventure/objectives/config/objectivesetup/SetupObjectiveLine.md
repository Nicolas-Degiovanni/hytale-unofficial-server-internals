---
description: Architectural reference for SetupObjectiveLine
---

# SetupObjectiveLine

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.objectivesetup
**Type:** Configuration Object

## Definition
```java
// Signature
public class SetupObjectiveLine extends ObjectiveTypeSetup {
```

## Architecture & Concepts
SetupObjectiveLine is a concrete implementation of the **Strategy Pattern**, represented by its parent class ObjectiveTypeSetup. Its primary role is to act as a configuration-driven factory for initiating a sequence of related objectives, known as an "Objective Line".

This class is not a long-lived service. Instead, it is a transient data-holding object deserialized from game asset files by the Hytale **Codec** system. It serves as a critical bridge between static game data (an ObjectiveLineAsset identified by an ID) and the live, dynamic game state managed by the ObjectivePlugin service. Its sole purpose is to translate a simple asset identifier into a complex, stateful Objective instance within the game world, assigning it to a specific set of players.

The static CODEC field is central to its design. It defines the deserialization contract, including validation rules that ensure the referenced ObjectiveLineAsset exists before the configuration is considered valid. This pre-validation is crucial for preventing runtime errors due to missing or malformed game data.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the Hytale `Codec` system during the deserialization of a parent configuration object, such as a quest definition or a zone trigger. It is never created manually using the `new` keyword.
-   **Scope:** The object's scope is extremely short-lived. It typically exists only for the duration of a single method call. Once its `setup` method is invoked, the core system no longer holds a reference to the instance, making it immediately eligible for garbage collection.
-   **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. There are no manual destruction or resource release methods.

## Internal State & Concurrency
-   **State:** The internal state consists of a single field, `objectiveLineId`. This state is **effectively immutable** after deserialization. The CODEC sets this value upon creation, and no public methods exist to modify it.
-   **Thread Safety:** This class is inherently **thread-safe** due to its immutable state. However, the methods it invokes are not. The `setup` method modifies global game state via the ObjectivePlugin and **must** be called from the main server thread to prevent race conditions and state corruption. Accessing the ObjectiveLineAsset map may also have its own threading constraints.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getObjectiveIdToStart() | String | O(1) | Retrieves the asset ID of the *first* objective in the configured line. Returns null if the ObjectiveLineAsset is not found. |
| setup(players, world, marker, store) | Objective | O(N) | The primary entry point. Delegates to the ObjectivePlugin to instantiate and begin the objective line for the specified players. |

## Integration Patterns

### Standard Usage
This object is not used directly. Instead, it is configured within a larger asset file and invoked polymorphically by a system that manages objectives. The system retrieves an instance of ObjectiveTypeSetup and calls its `setup` method, unaware of the concrete implementation.

```java
// Assume 'someTriggerConfig' is an object deserialized from an asset file.
// The 'objectiveSetup' field is an instance of SetupObjectiveLine.
ObjectiveTypeSetup objectiveSetup = someTriggerConfig.getObjectiveSetup();

// The system invokes setup at the appropriate time (e.g., when a player enters a zone).
// This call polymorphically dispatches to SetupObjectiveLine.setup().
objectiveSetup.setup(playersInZone, worldUUID, triggerMarkerUUID, entityStore);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new SetupObjectiveLine()`. The object must be created via the `CODEC` to ensure its internal state is correctly populated and validated against the game's asset database. Manual creation bypasses critical validation logic.
-   **State Mutation:** Do not use reflection or other means to modify the `objectiveLineId` after creation. The entire system relies on this ID being a faithful representation of the source asset data.

## Data Pipeline
SetupObjectiveLine acts as a command object within a larger data-to-action pipeline. It translates declarative configuration into an imperative action.

> Flow:
> Game Asset (JSON/HOCON) -> Codec Deserializer -> **SetupObjectiveLine Instance** -> `setup()` Invocation -> ObjectivePlugin -> New Objective created in World State

