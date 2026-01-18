---
description: Architectural reference for KillObjectiveTask
---

# KillObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.task
**Type:** Transient Component

## Definition
```java
// Signature
public abstract class KillObjectiveTask extends CountObjectiveTask implements KillTask {
```

## Architecture & Concepts
The KillObjectiveTask is an abstract base class that represents a server-side objective component for tracking player kills against a specified group of NPCs. It is a fundamental building block of the adventure mode quest system, translating raw combat events into meaningful objective progress.

Architecturally, this class functions as a reactive event processor. It does not actively poll game state. Instead, it is designed to be invoked by a higher-level system that manages entity death events. Its core responsibility is to filter these events based on criteria defined in its corresponding asset file, KillObjectiveTaskAsset. This data-driven design allows content creators to define complex "kill quests" without writing new code.

This class acts as a bridge between the core **Damage System** and the **Objective System**. It inherits its counting mechanism from CountObjectiveTask and adheres to the contract defined by the KillTask interface, ensuring it can be discovered and managed by the objective lifecycle manager.

### Lifecycle & Ownership
The lifecycle of a KillObjectiveTask instance is strictly managed by the server's objective framework and is tied to a parent Objective.

-   **Creation:** Instances are not created directly using the new keyword. They are instantiated and deserialized by the objective system via the provided BuilderCodec when a player accepts or progresses to a quest stage containing this task. The configuration is loaded from a corresponding KillObjectiveTaskAsset.
-   **Scope:** The instance exists for the duration that its parent Objective is active for a specific player or party. It holds the progress for that specific quest instance.
-   **Destruction:** The object is eligible for garbage collection once the parent Objective is completed, failed, or otherwise terminated. It has no explicit cleanup or teardown logic.

## Internal State & Concurrency
-   **State:** This class is stateful. It inherits a mutable counter from its parent, CountObjectiveTask, which tracks the number of qualifying kills. All other configuration, such as the target NPC group, is read from the immutable KillObjectiveTaskAsset and is considered static for the object's lifetime.
-   **Thread Safety:** This component is not thread-safe and is designed to be operated on by the main server game loop thread. All invocations of its methods must be synchronized externally, typically by the event dispatcher that processes entity death. Unsynchronized access from multiple threads will lead to race conditions when updating the completion count.

## API Surface
The public API is minimal, exposing only the necessary contract for the objective system to drive its logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| checkKilledEntity(store, npcRef, objective, npc, info) | void | O(1) | Core logic handler. Validates if a given death event contributes to the objective's progress. Throws IllegalArgumentException if the NPC group in the asset is unknown. |
| getAsset() | KillObjectiveTaskAsset | O(1) | Retrieves the immutable asset data that configures this task instance. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class directly in Java code. Instead, they define a KillObjectiveTaskAsset in a content pack. The objective system automatically instantiates and manages the task.

The following example is a conceptual representation of how an event handler in the objective system would dispatch a death event to this task.

```java
// Conceptual: Inside an ObjectiveManager listening for entity death events
void onEntityKilled(NPCEntity killedNpc, Damage damageInfo) {
    // Find the objective this NPC might be relevant for
    Objective activeObjective = getActiveObjectiveForPlayer(damageInfo.getSource());

    // Iterate through the objective's tasks
    for (ObjectiveTask task : activeObjective.getTasks()) {
        if (task instanceof KillTask) {
            // Dispatch the event to any task that handles kills
            ((KillTask) task).checkKilledEntity(
                entityStore,
                killedNpc.getRef(),
                activeObjective,
                killedNpc,
                damageInfo
            );
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new KillObjectiveTask(). The object is intrinsically linked to its asset data and must be created by the engine's codec and objective systems to function correctly.
-   **Bypassing Validation:** Do not call increaseTaskCompletion directly. This would bypass the critical validation logic in checkKilledEntity, potentially allowing players to get credit for killing the wrong NPC or for kills made by other entities.
-   **State Manipulation:** Avoid attempting to modify the state of the underlying KillObjectiveTaskAsset at runtime. These assets are intended to be read-only after engine startup.

## Data Pipeline
The KillObjectiveTask sits in the middle of a data flow that transforms a low-level game event into a high-level player-facing outcome.

> Flow:
> Player deals fatal damage -> Damage System creates **Damage** object -> Entity Death Event is fired -> Objective Manager receives event -> Manager invokes **checkKilledEntity** on active **KillObjectiveTask** -> Task validates and calls **increaseTaskCompletion** -> Parent **Objective** state is updated -> Objective Update Event is fired -> UI System updates player's quest tracker

