---
description: Architectural reference for CountObjectiveTask
---

# CountObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Transient State Object

## Definition
```java
// Signature
public abstract class CountObjectiveTask extends ObjectiveTask {
```

## Architecture & Concepts
The **CountObjectiveTask** is an abstract base class that represents the runtime state for any objective task based on counting. It embodies a critical design pattern within the Adventure Mode system: the separation of static configuration from dynamic, in-game state.

The static configuration is defined by a corresponding **CountObjectiveTaskAsset**, which specifies the target count, description keys, and other immutable properties. This **CountObjectiveTask** instance, in contrast, holds the mutable *current progress* (the count) for a specific objective assigned to a player or group.

Architecturally, it functions as a state machine managed by a parent **Objective** instance. Its sole responsibility is to track a numerical value, check it against a target, and report completion status up to its parent. It is not a self-contained system; it is a component part of a larger objective composition, acting as a leaf node in the objective's task hierarchy.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated by the **Objective** system when an objective is activated. The parent **Objective** reads its corresponding asset, identifies the required tasks, and constructs the necessary **CountObjectiveTask** instances to track runtime progress.

- **Scope:** The lifecycle of a **CountObjectiveTask** is strictly bound to its parent **Objective**. It persists as long as the objective is active for a player.

- **Destruction:** The object is eligible for garbage collection once the parent **Objective** is completed, abandoned, or otherwise terminated. There is no explicit destruction method; its memory is reclaimed when all references from the managing **Objective** are released.

## Internal State & Concurrency
- **State:** This class is fundamentally mutable. Its primary state is the integer field **count**, which is frequently updated by game logic. It also inherits a boolean **complete** flag from its parent, **ObjectiveTask**.

- **Thread Safety:** **WARNING:** This class is not thread-safe and must not be accessed from any thread other than the main server thread. All public methods that mutate state, such as **increaseTaskCompletion**, are designed to be called synchronously within the server's primary game loop. Unsynchronized access will lead to race conditions, data corruption, and inconsistent objective state.

## API Surface
The public API is designed for state mutation and progression, triggered by external game events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| increaseTaskCompletion(store, ref, qty, objective) | void | O(1) | Atomically increases the internal count. This is the primary method for progressing the task. Triggers network updates and completion checks. |
| setTaskCompletion(store, ref, qty, objective) | void | O(1) | Atomically sets the internal count to a specific value. Triggers network updates and completion checks. |
| checkCompletion() | boolean | O(1) | Returns true if the current count meets or exceeds the target count defined in the asset. |
| getInfoMessage(objective) | Message | O(1) | Constructs a formatted message for UI display, typically showing progress like "5/10". |

## Integration Patterns

### Standard Usage
Interaction with this class should always be mediated through the parent **Objective**. Game systems (e.g., block break listeners, entity death handlers) should retrieve the active objective and then call the appropriate progression method on the task.

```java
// Example: A system that handles mob kills
// Assume 'playerObjective' is the currently active Objective for a player

// Find the correct task within the objective (logic not shown)
CountObjectiveTask killTask = playerObjective.findTask(CountObjectiveTask.class, "mob_kill_task");

if (killTask != null && !killTask.isComplete()) {
    // The 'store' and 'ref' are passed in from the server context
    killTask.increaseTaskCompletion(store, ref, 1, playerObjective);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use **new CountObjectiveTask()**. The class is abstract and its lifecycle is managed exclusively by the **Objective** system to ensure state consistency.

- **State Polling:** Do not repeatedly call **checkCompletion** in a tight loop. The class follows an event-driven pattern; logic should be triggered by the completion event propagated from the **updateTaskCompletion** method.

- **Direct Field Modification:** Modifying the public **count** field directly will bypass critical logic, including network synchronization and completion checks. Always use the provided mutator methods.

## Data Pipeline
The flow of data for a task update is unidirectional, originating from a game event and propagating to the client UI.

> Flow:
> Game Event (e.g., Mob Killed) -> Server-side Event Listener -> **CountObjectiveTask**.increaseTaskCompletion() -> **Objective**.markDirty() -> Network Packet Sent -> Client UI Receives Update

