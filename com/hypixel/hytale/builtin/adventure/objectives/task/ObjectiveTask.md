---
description: Architectural reference for ObjectiveTask
---

# ObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Stateful Component (Abstract Base Class)

## Definition
```java
// Signature
public abstract class ObjectiveTask implements NetworkSerializer<Objective, com.hypixel.hytale.protocol.ObjectiveTask> {
```

## Architecture & Concepts

The ObjectiveTask is the fundamental unit of work within the Hytale Adventure Objective system. It represents a single, stateful, and executable stage of a larger Objective. Each concrete implementation of this abstract class defines a specific goal for the player, such as "collect item X", "reach location Y", or "defeat entity Z".

This class is not a standalone entity; it is intrinsically owned and managed by a parent Objective. Its behavior and initial state are defined by a corresponding ObjectiveTaskAsset, which acts as a configuration blueprint loaded from game data.

Key architectural pillars of this system include:

*   **Transaction-Based World Interaction:** ObjectiveTasks do not manipulate the game world directly. Instead, they create and manage a set of TransactionRecords. These records represent reversible operations, such as spawning an entity or modifying a block. This transactional approach ensures that the world can be cleanly reverted to its previous state if the objective is abandoned or fails, preventing permanent, orphaned changes to the game state.

*   **Event-Driven Completion:** Each task instance maintains its own isolated EventRegistry. During its setup phase, it subscribes to specific game events relevant to its completion criteria. This decouples the task logic from the main game loop, allowing for an efficient, reactive design where the task only performs checks when a potentially relevant event occurs.

*   **State Synchronization:** As a server-side component, the ObjectiveTask is responsible for synchronizing its state with connected clients. It serializes its current status into UpdateObjectiveTask packets, ensuring that players' user interfaces accurately reflect their progress.

## Lifecycle & Ownership

The lifecycle of an ObjectiveTask is strictly controlled by its parent Objective and is designed to be ephemeral, existing only for the duration that it is the active stage of a quest.

*   **Creation:** An ObjectiveTask is instantiated by its parent Objective when it becomes the active task. It is configured using an ObjectiveTaskAsset, which dictates its type and properties. The `CODEC` and `BASE_CODEC` static fields indicate that instances are also created via deserialization when loading a game state where an objective is in progress.

*   **Scope:** The object's lifetime is precisely scoped to its active period within the parent Objective. It begins with the `setup` method call and ends when it is either completed or reverted.

*   **Destruction:** Cleanup is an explicit and critical process. When a task is finished, one of three terminal methods must be called: `completeTransactionRecords`, `revertTransactionRecords`, or `unloadTransactionRecords`. Each of these methods performs the following critical cleanup operations:
    1.  Finalizes or reverts all associated TransactionRecords.
    2.  Shuts down its dedicated EventRegistry to prevent resource leaks.
    3.  Unregisters its ObjectiveTaskRef from the global ObjectiveDataStore, removing it from the system.

    **WARNING:** Failure to call one of these terminal methods will result in severe resource leaks, including dangling event listeners and orphaned game entities.

## Internal State & Concurrency

*   **State:** The ObjectiveTask is highly mutable. Its core state includes the `complete` flag, a list of active `MapMarker` objects for the client, and arrays of `TransactionRecord` objects representing its impact on the world. This state is modified throughout its lifecycle, particularly during the `setup` and `complete` phases.

*   **Thread Safety:** This class is **not thread-safe**. All interactions with an ObjectiveTask instance must be performed on the main server thread. The internal use of `CopyOnWriteArrayList` within its EventRegistry is for safe event dispatching, not for enabling concurrent modification of the task's state from external threads. Unsynchronized access will lead to race conditions and unpredictable game state corruption.

## API Surface

The public API is designed for control by a parent Objective. Direct manipulation by other systems is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup(objective, store) | TransactionRecord[] | O(N) | Initializes the task. Creates world transactions, registers event listeners, and adds map markers. Throws IllegalStateException if called more than once. |
| checkCompletion() | boolean | O(1) | Abstract method. Concrete implementations use this to define the logic for task completion. |
| complete(objective, accessor) | void | O(P) | Marks the task as complete, notifies participants, and triggers the cleanup of all associated resources and transactions. |
| revertTransactionRecords() | void | O(T) | Reverts all world changes made by the task and performs a full resource cleanup. Used when an objective is abandoned. |
| sendUpdateObjectiveTaskPacket(objective) | void | O(P) | Serializes the task's current state and broadcasts it to all active participants in the objective. |
| areTaskConditionsFulfilled(...) | boolean | O(C) | Evaluates a set of prerequisite conditions defined in the asset. Must return true before the task can be completed. |

*N: Number of initial setup operations (e.g., markers, transactions). P: Number of objective participants. T: Number of transactions. C: Number of task conditions.*

## Integration Patterns

### Standard Usage

The ObjectiveTask is designed to be driven by its parent Objective in a well-defined sequence.

```java
// Context: Within an Objective class, advancing to a new task.

// 1. Retrieve the new task, typically from a list.
ObjectiveTask currentTask = this.tasks.get(nextTaskIndex);

// 2. Initialize the task. This registers listeners and modifies the world.
TransactionRecord[] records = currentTask.setup(this, entityStore);
if (TransactionUtil.hasFailed(records)) {
    // Handle setup failure...
    return;
}

// 3. The game loop and event system will now interact with the task.
//    Eventually, checkCompletion() will return true.

// 4. Once completion is detected, finalize the task.
if (currentTask.checkCompletion() && currentTask.areTaskConditionsFulfilled(...)) {
    currentTask.consumeTaskConditions(...);
    currentTask.complete(this, entityStoreAccessor);
    // Proceed to the next task...
}
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not manually instantiate concrete subclasses of ObjectiveTask. They must be created and managed by the parent Objective, typically through deserialization of an ObjectiveAsset.
*   **Re-using Instances:** An ObjectiveTask instance is single-use. Once `complete`, `revert`, or `unload` has been called, the instance is considered defunct and must be discarded. Do not attempt to call `setup` again.
*   **External State Modification:** Do not modify the internal state (e.g., the `complete` flag) of a task from outside the class. Use the provided public methods like `complete` to ensure the entire lifecycle and cleanup process is correctly executed.
*   **Ignoring Setup Failures:** The `setup` method can fail, returning a failed TransactionRecord. Ignoring this failure will lead to an objective being stuck in an invalid or incomplete state.

## Data Pipeline

The ObjectiveTask acts as a state machine that processes game events and produces world changes and network updates.

> Flow:
> ObjectiveAsset (Data) -> Deserializer -> **ObjectiveTask** (Instantiation) -> `setup()` call -> World State Change (via TransactionRecords)
>
> Game Event -> EventRegistry -> **ObjectiveTask** -> `checkCompletion()` evaluation
>
> Completion -> `complete()` call -> **ObjectiveTask** -> `UpdateObjectiveTask` Packet -> Network -> Client UI Update

