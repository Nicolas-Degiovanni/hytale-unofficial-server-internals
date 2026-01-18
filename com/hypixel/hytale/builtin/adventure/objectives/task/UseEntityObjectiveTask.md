---
description: Architectural reference for UseEntityObjectiveTask
---

# UseEntityObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Transient State Object

## Definition
```java
// Signature
public class UseEntityObjectiveTask extends CountObjectiveTask {
```

## Architecture & Concepts

The UseEntityObjectiveTask is a concrete implementation of a server-side objective task, designed to track a player's interaction with a specific set of unique entities. It functions as a stateful component within a parent Objective, representing a single, measurable step towards the objective's completion.

Architecturally, this class is a data-driven state machine. Its behavior and configuration are defined by a corresponding **UseEntityObjectiveTaskAsset**, which specifies parameters like the target count and dialog options upon completion. It extends **CountObjectiveTask**, inheriting the fundamental logic for counting towards a target number. Its primary contribution is the addition of a uniqueness constraint: it only increments its count for the *first* interaction with any given entity, preventing a player from repeatedly using the same entity to complete the task.

This class integrates tightly with the **ObjectivePlugin** and **ObjectiveDataStore**, which act as central registries. During its setup phase, the task registers itself with the data store, allowing the server's event system to route relevant entity interaction events to the correct task instance for processing.

## Lifecycle & Ownership

-   **Creation:** An instance of UseEntityObjectiveTask is not created directly. It is instantiated by the objective framework when a parent **Objective** is activated for a player. The Objective's configuration, loaded from an asset, dictates the creation and configuration of its child tasks.
-   **Scope:** The object's lifetime is strictly bound to its parent Objective instance. It persists as long as the objective is active, and its state (specifically, the set of interacted entity UUIDs) is serialized as part of the parent objective's data.
-   **Destruction:** The object is eligible for garbage collection once its parent Objective is completed, abandoned, or otherwise removed from the game state. There is no explicit destruction method; its lifecycle is managed entirely by its owner.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its core internal state is the **npcUUIDs** set, which stores the UUIDs of entities that have already been used to progress the task. This set grows dynamically during the task's lifecycle and is the key to enforcing the uniqueness constraint.
-   **Thread Safety:** **WARNING:** This class is not thread-safe. All methods that mutate its internal state, particularly increaseTaskCompletion, are designed to be called exclusively from the main server game thread (the world tick). The internal use of a standard HashSet without any synchronization mechanisms makes it vulnerable to race conditions and corruption if accessed from multiple threads. All interactions must be marshaled to the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(P) | Initializes the task. Registers it with the ObjectiveDataStore for each active player (P). Generates a transaction record for state persistence. |
| increaseTaskCompletion(store, ref, qty, objective, playerRef, npcUUID) | boolean | O(1) | The primary logic handler. Attempts to progress the task using the provided npcUUID. Returns false if the entity has already been used. |

## Integration Patterns

### Standard Usage

A developer does not typically interact with this class directly. The server's objective system orchestrates its use. An event listener for entity interactions would acquire the player's active objectives and delegate the event to the appropriate task.

```java
// Hypothetical event handler within the objective system
void onPlayerUseEntity(PlayerRef playerRef, UUID usedEntityUUID) {
    ObjectiveDataStore dataStore = ObjectivePlugin.get().getObjectiveDataStore();
    
    // Find tasks registered for this player and entity type
    for (Objective objective : dataStore.getActiveObjectivesForPlayer(playerRef.getUUID())) {
        for (ObjectiveTask task : objective.getActiveTasks()) {
            if (task instanceof UseEntityObjectiveTask) {
                UseEntityObjectiveTask useTask = (UseEntityObjectiveTask) task;
                
                // The system calls this to process the event
                boolean progressed = useTask.increaseTaskCompletion(
                    world.getStore(), 
                    playerRef.getRef(), 
                    1, 
                    objective, 
                    playerRef, 
                    usedEntityUUID
                );

                if (progressed) {
                    // Trigger state save, notify player, etc.
                }
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new UseEntityObjectiveTask()`. The object is useless without being properly configured from an asset and managed by a parent Objective.
-   **External State Mutation:** Do not directly access and modify the public `npcUUIDs` set. Doing so would bypass all game logic, including completion checks, player feedback messages, and dialog triggers.
-   **Asynchronous Processing:** Never call `increaseTaskCompletion` from a network thread, database callback, or any other thread besides the main server thread. This will lead to state corruption.

## Data Pipeline

The flow of data for a typical interaction event demonstrates this class's position within the adventure mode systems.

> Flow:
> Player Input (Interact Key) -> Server Network Packet -> Entity Interaction Game Event -> **Objective Event Listener** -> ObjectiveDataStore Lookup -> **UseEntityObjectiveTask.increaseTaskCompletion()** -> Internal State Update (npcUUIDs set) -> Objective Progress Check -> (On Completion) -> Player Message & DialogPage UI Event

