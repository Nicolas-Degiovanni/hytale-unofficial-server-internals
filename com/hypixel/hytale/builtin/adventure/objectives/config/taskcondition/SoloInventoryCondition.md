---
description: Architectural reference for SoloInventoryCondition
---

# SoloInventoryCondition

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.taskcondition
**Type:** Configuration Asset

## Definition
```java
// Signature
public class SoloInventoryCondition extends TaskConditionAsset {
```

## Architecture & Concepts
The SoloInventoryCondition is a data-driven configuration asset, not a service or a manager. It represents a specific, concrete rule within the Adventure Mode objective system. Its primary function is to determine if a task's completion criteria, related to a single player's inventory, have been met.

Architecturally, this class is a terminal node in the engine's deserialization pipeline. The static **CODEC** field is the most critical feature; it allows the Hytale engine to read a plain-text definition (e.g., from a JSON file) and instantiate a fully configured SoloInventoryCondition object in memory. This pattern decouples game logic from game design, enabling designers to create complex inventory-based quest objectives without writing any Java code.

It acts as a predicate, a function that evaluates the state of a player entity and returns a boolean result. It encapsulates the logic for checking item types, quantities, and placement (e.g., held in hand vs. anywhere in inventory).

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** system during the asset loading phase. The engine reads an objective or task definition from a data file and uses the static CODEC to deserialize the configuration into a new SoloInventoryCondition object. Direct instantiation is an anti-pattern.
- **Scope:** The object's lifetime is bound to its parent asset, typically an objective or task configuration. It is loaded into memory when the adventure content is initialized and persists as long as that content is active. The object's internal state is effectively immutable after creation.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded, for example, when a server shuts down or a world's adventure data is flushed.

## Internal State & Concurrency
- **State:** The state of a SoloInventoryCondition is defined by its fields: *blockTypeOrTagTask*, *quantity*, *consumeOnCompletion*, and *holdInHand*. This state is populated once during deserialization and is considered **immutable** for the object's entire lifecycle. There are no public methods to mutate this state.
- **Thread Safety:** This class is **conditionally thread-safe**. The object's internal state is safe to read from any thread due to its immutability. However, its core methods, *isConditionFulfilled* and *consumeCondition*, operate on mutable game state (the Player component and its Inventory).

    **WARNING:** These methods **must** be called from the main server game-tick thread. Invoking them from an asynchronous task or a different thread will lead to race conditions, data corruption, and server instability. The responsibility for ensuring single-threaded access to the player entity lies with the calling system, which is the objective-processing engine.

## API Surface
The public API is designed for evaluation and execution by the objective system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isConditionFulfilled(accessor, ref, players) | boolean | O(N) | Evaluates the player's inventory against the configured rules. N is the number of slots in the inventory. Returns false if the target entity is not a Player. |
| consumeCondition(accessor, ref, players) | void | O(N) | If *consumeOnCompletion* is true, this method removes the required items from the player's inventory. It is a no-op if the flag is false. |
| getBlockTypeOrTagTask() | BlockTagOrItemIdField | O(1) | Returns the configuration object defining the target item or item tag. |
| getQuantity() | int | O(1) | Returns the required quantity of the target item. |
| isConsumeOnCompletion() | boolean | O(1) | Returns true if the items should be removed upon task completion. |
| isHoldInHand() | boolean | O(1) | Returns true if the player must be holding the item for the condition to be met. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most systems. It is invoked by the server's core objective engine, which iterates through active tasks and evaluates their conditions for each relevant player.

```java
// PSEUDO-CODE: Illustrates how the objective engine uses this asset.
// This logic would exist within a higher-level objective processing system.

// 'condition' is an instance of SoloInventoryCondition loaded from a quest asset file.
TaskConditionAsset condition = currentTask.getCondition();

// The engine provides the ComponentAccessor and a reference to the player entity.
if (condition.isConditionFulfilled(world.getComponentAccessor(), playerRef, objectivePlayers)) {
    // The condition is met; advance the objective.
    objective.advanceState();

    // Trigger the consumption logic if configured to do so.
    condition.consumeCondition(world.getComponentAccessor(), playerRef, objectivePlayers);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SoloInventoryCondition()`. This bypasses the data-driven design and results in an unconfigured, useless object. All instances must be created from data files via the engine's CODEC system.
- **State Mutation:** Do not use reflection to modify the fields of a loaded instance. The asset system assumes these objects are immutable. Changing state at runtime can lead to inconsistent behavior for all players engaged in the same quest.
- **Asynchronous Evaluation:** Do not call *isConditionFulfilled* or *consumeCondition* from a separate thread. Player inventory and component data are not thread-safe and must only be accessed from the main server thread.

## Data Pipeline
The flow of data from design-time configuration to run-time evaluation is central to this class's purpose.

> Flow:
> Adventure Mode JSON File -> Hytale Asset Loader -> **SoloInventoryCondition.CODEC** -> In-Memory **SoloInventoryCondition** Object -> Objective Engine Evaluation -> Player Inventory Check -> Objective State Update

