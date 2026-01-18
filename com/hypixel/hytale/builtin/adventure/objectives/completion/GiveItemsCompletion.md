---
description: Architectural reference for GiveItemsCompletion
---

# GiveItemsCompletion

**Package:** com.hypixel.hytale.builtin.adventure.objectives.completion
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class GiveItemsCompletion extends ObjectiveCompletion {
```

## Architecture & Concepts
The GiveItemsCompletion class is a server-side implementation of the **Strategy Pattern**, defining a specific outcome for an Adventure Mode objective. Its sole responsibility is to grant one or more items to all participants of an objective when that objective is successfully completed.

This class acts as a transactional bridge between the high-level Objective system and the low-level Inventory and Item systems. It is entirely data-driven, configured by a corresponding GiveItemsCompletionAsset. This asset dictates which items are granted by referencing a drop list ID, decoupling the completion logic from specific item definitions.

Upon execution, this component queries the core ItemModule to resolve the drop list into concrete ItemStacks, then interacts directly with the Entity Component System (ECS) to modify the inventory of each participating entity. It is a critical component for implementing quest rewards and loot distribution within the game's adventure mode.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's Objective system when an objective configured with a `giveItems` completion type is triggered. It is never created manually by game logic developers. The instance is created just-in-time to handle the completion event.
- **Scope:** The object's lifetime is extremely short, scoped exclusively to the duration of the `handle` method's execution. It is a stateless, single-use object.
- **Destruction:** The object holds no persistent state or external references beyond its execution scope. It becomes eligible for garbage collection immediately after the `handle` method returns.

## Internal State & Concurrency
- **State:** Effectively immutable. The class holds a single read-only reference to its configuration asset. The `handle` method does not modify any internal fields; all state changes are applied to external systems like player inventories and objective history.
- **Thread Safety:** **Not Thread-Safe**. This class performs direct, unsynchronized modifications to the EntityStore and player inventories. It **must** be executed on the main thread of the world in which the objective exists. Calling the `handle` method from any other thread will result in race conditions, data corruption, and server crashes.

## API Surface
The public contract is minimal, consisting only of the polymorphic `handle` method required by the ObjectiveCompletion interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(objective, accessor) | void | O(N * M) | Executes the item reward logic. N is the number of participants; M is the number of items granted. Throws NullPointerException if arguments are null. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is configured declaratively within an objective's asset definition. The system instantiates and calls it automatically.

The following example is a conceptual representation of how the engine invokes this component.

```java
// ENGINE-INTERNAL CODE: DO NOT REPLICATE
// The Objective system receives a completion event and dispatches
// to the appropriate handler based on the objective's asset.

Objective objectiveToComplete = ...;
ObjectiveCompletion completionHandler = objectiveToComplete.getCompletionHandler(); // Returns an instance of GiveItemsCompletion

// The engine provides the necessary ECS accessor for the world's thread
ComponentAccessor<EntityStore> accessor = ...;
completionHandler.handle(objectiveToComplete, accessor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new GiveItemsCompletion(asset)`. The lifecycle is managed entirely by the Objective system to ensure correct state and context.
- **Manual Invocation:** Do not acquire an instance and call `handle` manually. This bypasses the state transitions and validation logic of the parent Objective, potentially leading to duplicate rewards or inconsistent quest states.
- **Cross-Thread Execution:** Invoking `handle` from a worker thread, network thread, or any context other than the target world's main update loop is a critical stability risk.

## Data Pipeline
The flow of data and control for this component is linear and unidirectional, originating from a game event and resulting in a network packet sent to the client.

> Flow:
> Objective Completion Event -> Objective System -> **GiveItemsCompletion.handle()** -> ItemModule -> Player Inventory Component (State Change) -> ObjectiveHistoryData (State Change) -> NotificationUtil -> Network Packet -> Client UI Notification<ctrl63>

