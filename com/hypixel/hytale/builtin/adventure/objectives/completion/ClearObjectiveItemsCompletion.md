---
description: Architectural reference for ClearObjectiveItemsCompletion
---

# ClearObjectiveItemsCompletion

**Package:** com.hypixel.hytale.builtin.adventure.objectives.completion
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class ClearObjectiveItemsCompletion extends ObjectiveCompletion {
```

## Architecture & Concepts
ClearObjectiveItemsCompletion is a concrete implementation of the **Strategy Pattern**, designed to execute a specific cleanup task upon the successful completion of an in-game objective. Its sole responsibility is to scan the inventories of all participating entities and remove any items specifically tagged as belonging to the completed objective.

This component operates exclusively on the server-side as part of the Adventure Mode objective system. It identifies "quest items" not by their type or name, but by a unique identifier (UUID) stored in the item's metadata. This powerful, data-driven approach decouples the objective logic from specific item definitions, allowing any item to be dynamically designated as a quest item.

The class acts as a final-state mutator in the objective lifecycle, ensuring that temporary items used to fulfill a quest do not persist in the player's inventory after the objective is finished.

## Lifecycle & Ownership
- **Creation:** An instance of ClearObjectiveItemsCompletion is created by the objective system's asset loader. It is instantiated when an objective's configuration, defined in a ClearObjectiveItemsCompletionAsset, is deserialized and loaded into the game. Developers do not instantiate this class directly.
- **Scope:** The object's lifetime is tightly bound to its parent Objective instance. It is created when the objective is loaded and remains in memory as part of the objective's definition.
- **Destruction:** The instance is eligible for garbage collection when its parent Objective is unloaded or completed and subsequently discarded by the objective management system. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its behavior is determined by the immutable ClearObjectiveItemsCompletionAsset provided during construction. The core `handle` method is a pure function of its inputs (the objective and the world state) and does not modify its own internal state.
- **Thread Safety:** **This class is not thread-safe and must not be used outside the main server thread.** The `handle` method directly accesses and mutates entity inventory state via a ComponentAccessor. Invoking it from any other thread will lead to race conditions, data corruption, and server instability. All objective logic is designed to be executed synchronously within the server's primary game loop tick.

## API Surface
The public contract is minimal, consisting only of the overridden `handle` method which is the entry point for its logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(objective, componentAccessor) | void | O(N * M) | Executes the item-clearing logic. Iterates through N participants and M inventory slots for each. This operation is blocking and must only be called from the main server thread. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is not intended. Its usage is declared within an objective's asset configuration file. The game engine's objective system is responsible for instantiating and invoking it at the correct point in the objective's lifecycle.

A conceptual asset configuration might look like this:

```yaml
# Example Objective Asset
objectiveId: "gather_sacred_stones"
# ... other objective properties
completionActions: [
  {
    type: "ClearObjectiveItems" # This string maps to ClearObjectiveItemsCompletionAsset
    # No further properties needed for this specific action
  }
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ClearObjectiveItemsCompletion()`. The system is entirely data-driven, and manual instantiation bypasses the asset configuration that links it to the correct objective.
- **Manual Invocation:** Do not call the `handle` method directly. This circumvents the objective's state machine and could clear items from a player's inventory prematurely or under incorrect conditions.
- **Stateful Extensions:** Do not extend this class to add mutable state. The objective system assumes completion handlers are stateless and reusable.

## Data Pipeline
The data flow for this component is triggered by a state change within the objective system. It reads from the world state and writes back to it by mutating entity inventories.

> Flow:
> Objective State Machine -> `Objective.onComplete()` Event -> **ClearObjectiveItemsCompletion.handle()** -> Iterates Entity Participants -> Reads Entity Inventory -> Removes ItemStacks with matching metadata -> Entity Inventory State is Mutated

