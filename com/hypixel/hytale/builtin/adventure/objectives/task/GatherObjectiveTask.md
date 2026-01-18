---
description: Architectural reference for GatherObjectiveTask
---

# GatherObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Transient State Object

## Definition
```java
// Signature
public class GatherObjectiveTask extends CountObjectiveTask {
```

## Architecture & Concepts
The GatherObjectiveTask is a concrete implementation of an objective task within the Adventure Mode Objective System. Its sole responsibility is to track the quantity of a specific item, or a class of items defined by a tag, across the combined inventories of all players participating in the parent Objective.

This class is a server-side component that operates in a highly efficient, event-driven manner. Rather than continuously polling player inventories, it subscribes to the LivingEntityInventoryChangeEvent. This reactive pattern ensures that the task's state is only re-evaluated when a relevant game event occurs, minimizing performance overhead.

It serves as a bridge between the game's event system and the state machine of an Objective. Its logic is tightly coupled with the server's core data structures, including the Universe, World, and EntityStore, from which it reads player and inventory data.

## Lifecycle & Ownership
The lifecycle of a GatherObjectiveTask instance is strictly controlled by the objective system and is not intended for manual management.

-   **Creation:** An instance is created by its parent Objective when that objective becomes active. The Objective framework reads the task's configuration from a GatherObjectiveTaskAsset and instantiates this class accordingly.
-   **Scope:** The object persists only for the duration of its parent Objective's active state. Its lifetime is ephemeral and tied directly to the progress of a specific quest or goal.
-   **Destruction:** The instance is marked for garbage collection when the parent Objective is completed, failed, or otherwise unloaded. The Objective is responsible for ensuring that all event listeners registered by this task are properly deregistered to prevent memory leaks.

## Internal State & Concurrency
-   **State:** This object is highly mutable. It inherits stateful fields like *count* and *complete* from its parent, CountObjectiveTask. This state is updated dynamically based on in-game events. It does not cache inventory data, re-calculating it on each relevant event to ensure accuracy.
-   **Thread Safety:** This class is **not thread-safe** and must be accessed only from its owning world's main thread.
    -   The core logic within the LivingEntityInventoryChangeEvent listener is explicitly scheduled to run on the world's thread via `world.execute`. This is a critical concurrency control pattern that prevents race conditions when accessing shared data like the EntityStore.
    -   **WARNING:** Any attempt to mutate or read the state of this object from an external thread will result in undefined behavior, data corruption, or server instability.

## API Surface
The public API is minimal, as this class is primarily intended for internal use by the objective system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(P\*I) | **Internal Lifecycle Method.** Initializes the task. Performs an initial inventory scan and registers the event listener. Complexity depends on player count (P) and average inventory size (I). |
| getAsset() | GatherObjectiveTaskAsset | O(1) | Returns the immutable asset configuration for this task. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. Instead, it is configured declaratively within an objective asset file, which is then loaded by the game's objective system.

```json
// Example pseudo-asset configuration
{
  "id": "collect_10_wood",
  "tasks": [
    {
      "type": "GATHER",
      "asset": "hytale:gather_objective_task_asset",
      "properties": {
        "blockTagOrItemIdField": {
          "tag": "hytale:wood_logs"
        },
        "count": 10
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new GatherObjectiveTask()`. The objective framework is solely responsible for its lifecycle. Manual creation will result in a non-functional task that is not registered with any system.
-   **Manual State Mutation:** Do not modify the public *count* or *complete* fields directly. The internal event listener is the single source of truth for state transitions. External modifications will be overwritten and will break objective logic.
-   **Listener Mismanagement:** Do not attempt to manually register or unregister event listeners for this object. The `setup0` method returns a transaction record that the objective system uses to manage listener registration and cleanup automatically.

## Data Pipeline
The flow of data and control for this component is event-driven and follows a precise, thread-safe sequence.

> Flow:
> Player Inventory changes -> Server fires **LivingEntityInventoryChangeEvent** -> Event Bus dispatches event to the registered listener in GatherObjectiveTask -> The listener's lambda captures the event and schedules a new task on the **World's main thread** -> The scheduled task executes `countObjectiveItemInInventories` across all relevant players -> The result is passed to `setTaskCompletion`, updating the task's internal state -> The parent Objective polls the task's *isComplete* status on its next tick.

