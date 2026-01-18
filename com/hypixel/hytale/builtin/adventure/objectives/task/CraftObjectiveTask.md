---
description: Architectural reference for CraftObjectiveTask
---

# CraftObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Stateful Component

## Definition
```java
// Signature
public class CraftObjectiveTask extends CountObjectiveTask {
```

## Architecture & Concepts
The **CraftObjectiveTask** is a specialized, event-driven component within the Adventure Mode Objective System. It represents a concrete, measurable goal for a player: to craft a specific item a certain number of times.

Architecturally, this class acts as a reactive listener that bridges the core gameplay mechanics (crafting) with the abstract objective framework. It subscribes to server-wide game events and filters them to determine if they are relevant to its specific goal. This design decouples the objective tracking logic from the inventory and crafting systems; the crafting system does not need to know that an objective is listening.

This component is entirely data-driven, configured by a corresponding **CraftObjectiveTaskAsset**. The asset defines the target item and the required quantity, allowing designers to create varied crafting quests without writing new code.

## Lifecycle & Ownership
The lifecycle of a **CraftObjectiveTask** is strictly managed by its parent **Objective** and the world it is active within.

-   **Creation:** Instances are not created manually. They are deserialized and instantiated by the objective management system when an **Objective** containing a **CraftObjectiveTaskAsset** is loaded and activated for a player or group.
-   **Scope:** The instance exists for the duration that its parent **Objective** is active within a specific world. Its primary setup logic, which registers an event listener, is executed once the objective begins.
-   **Destruction:** The component is marked for garbage collection when the parent **Objective** is completed, failed, or otherwise unloaded. The **RegistrationTransactionRecord** returned by the setup method ensures that its associated **PlayerCraftEvent** listener is properly deregistered from the event bus, preventing memory leaks and phantom updates.

## Internal State & Concurrency
-   **State:** This class is stateful. It inherits a mutable counter from its parent, **CountObjectiveTask**, which tracks the number of successful crafts. It also holds an immutable reference to its configuration asset, **CraftObjectiveTaskAsset**.
-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** All state modifications are designed to be executed exclusively on the main server thread for the world in which the task is active. The event listener lambda is invoked by the world's event bus, which guarantees single-threaded execution within the game tick, preventing race conditions on the completion counter.

## API Surface
The public API is minimal, as the component is primarily controlled by the objective system through its protected lifecycle methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(1) | Registers an event listener for **PlayerCraftEvent**. This is the core activation method and is called by the parent **Objective**. |
| getAsset() | CraftObjectiveTaskAsset | O(1) | Returns the data asset that configures this task's behavior, such as the target item ID. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly. Instead, they define a **CraftObjectiveTaskAsset** in JSON or HOCON. The objective system handles the instantiation and lifecycle automatically. The `setup0` method is invoked internally by the parent **Objective** upon activation.

```java
// PSEUDOCODE: How the Objective system uses this class internally

// 1. An Objective is activated for a player in a given world.
Objective parentObjective = ...;
World activeWorld = ...;

// 2. The Objective iterates through its tasks and sets them up.
for (ObjectiveTask task : parentObjective.getTasks()) {
    if (task instanceof CraftObjectiveTask) {
        // The system invokes setup0, which registers the craft listener.
        task.setup0(parentObjective, activeWorld, entityStore);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CraftObjectiveTask()`. The object is useless without being managed by a parent **Objective** and its corresponding asset. Doing so will result in a component that does not listen to any events and cannot be progressed.
-   **Manual State Mutation:** Do not manually call `increaseTaskCompletion` or otherwise modify the internal counter. Progress should only be driven by in-game **PlayerCraftEvent** triggers to ensure quest logic remains consistent with player actions.
-   **Cross-Thread Access:** Do not store a reference to this task and attempt to read its progress from another thread. This will lead to thread-safety violations and unpredictable state.

## Data Pipeline
The data flow for this component is linear and event-driven, originating from a player action and culminating in a state update.

> Flow:
> Player performs a craft action in the UI -> Server validates and executes the craft -> **PlayerCraftEvent** is published to the World's Event Bus -> The listener registered by **CraftObjectiveTask** is invoked -> Listener logic filters the event by item ID and player -> **increaseTaskCompletion()** is called -> The parent **Objective** state is updated.

