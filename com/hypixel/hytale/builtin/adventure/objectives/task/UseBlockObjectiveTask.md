---
description: Architectural reference for UseBlockObjectiveTask
---

# UseBlockObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Stateful Task Object

## Definition
```java
// Signature
public class UseBlockObjectiveTask extends CountObjectiveTask {
```

## Architecture & Concepts
The UseBlockObjectiveTask is a concrete implementation within the Adventure Mode Objective framework. It represents a specific, measurable goal for a player: to use a designated block a certain number of times. This class is not a standalone service; it functions as a stateful, reactive component that is owned and managed by a parent Objective.

Its core architectural pattern is **event-driven**. Upon activation, it subscribes to a specific server-side event, LivingEntityUseBlockEvent. It then acts as a filter and processor for this event stream, checking if the event's context—specifically the player and the block type—matches the criteria defined in its configuration asset. When a match is found, it mutates its internal state (the count) and notifies the parent Objective of the progress.

This class is data-driven, configured entirely by a corresponding UseBlockObjectiveTaskAsset. This decouples the game logic from the specific quest data, allowing designers to create new objectives without modifying engine code.

### Lifecycle & Ownership
The lifecycle of a UseBlockObjectiveTask is strictly controlled by its parent Objective and the broader objective system.

-   **Creation:** An instance is created by the objective system when a parent Objective containing this task type is initialized for a player or group. The constructor is passed a UseBlockObjectiveTaskAsset, which dictates its behavior and completion criteria.
-   **Scope:** The object's lifetime is precisely bound to its parent Objective. It remains in memory and actively listens for events only while the parent Objective is active.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the parent Objective is completed, failed, or otherwise unloaded. The event listener registered during the setup phase is deregistered via the TransactionRecord system, preventing resource leaks. This cleanup is critical for server stability.

## Internal State & Concurrency
-   **State:** This class is stateful. It inherits a mutable integer counter from its parent, CountObjectiveTask, which tracks the number of times the target block has been used. It also holds an immutable reference to its configuration asset, UseBlockObjectiveTaskAsset.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. The design relies on the engine's single-threaded event processing model to guarantee safe state transitions. The event handler lambda in setup0 is always executed serially by the World's event bus. Any attempt to modify its state from an asynchronous task or different thread will result in race conditions and undefined behavior.

## API Surface
The primary interaction is through the setup0 method, which is part of the internal contract with the Objective system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(1) | Registers an event listener for LivingEntityUseBlockEvent. This is the activation point for the task. Returns a transaction record to allow for reliable deregistration and cleanup. |
| getAsset() | UseBlockObjectiveTaskAsset | O(1) | Retrieves the immutable configuration asset that defines this task's parameters, such as the target block type. |

## Integration Patterns

### Standard Usage
Developers and designers do not interact with this class directly in code. Its usage is declared within an objective's asset file. The engine's objective system is responsible for instantiating and managing it.

A designer would define a task in a JSON or HOCON asset file, which the server then parses to create the corresponding Java objects.

```json
// Conceptual Asset Definition
{
  "id": "example_quest",
  "tasks": [
    {
      "type": "hytale:use_block",
      "block": "hytale:crafting_table",
      "count": 5
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new UseBlockObjectiveTask() manually. The objective system manages its lifecycle. Manual creation will result in a non-functional, unmanaged object that does not listen to events and will not be cleaned up.
-   **Manual State Management:** Do not attempt to call internal methods like increaseTaskCompletion from outside the class. All state changes must be driven by the registered event handler to ensure game logic consistency.
-   **Reusing Instances:** Task instances are not reusable. They are bound to a single parent Objective instance and must be discarded along with it.

## Data Pipeline
The flow of data for this component is unidirectional and triggered by player interaction with the game world.

> Flow:
> Player interacts with a block -> Server fires LivingEntityUseBlockEvent -> World Event Bus dispatches event -> **UseBlockObjectiveTask listener** -> Filter event by player UUID and block type -> Call increaseTaskCompletion -> Parent Objective state is updated

