---
description: Architectural reference for BountyObjectiveTask
---

# BountyObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.task
**Type:** Transient State Object

## Definition
```java
// Signature
public class BountyObjectiveTask extends ObjectiveTask implements KillTask {
```

## Architecture & Concepts
The BountyObjectiveTask is a concrete implementation of the ObjectiveTask state machine, designed to manage a "kill a specific target" quest. It represents a single, stateful instance of this task type within a larger Objective.

Its primary architectural role is to bridge several core server systems:
*   **Objective System:** It functions as a stateful component managed by a parent Objective.
*   **NPC System:** It directly invokes the NPCPlugin to spawn the bounty target into the world.
*   **Event System:** It subscribes to a global KillTrackerResource, adopting an event-driven pattern to detect when its target has been eliminated.
*   **Persistence & Networking:** Through its static CODEC, the task's state (such as the target's UUID and completion status) can be serialized for game saves and deserialized to rehydrate the quest state. It also serializes its state into network packets for client-side UI updates.

This class is fundamentally data-driven, configured by a corresponding BountyObjectiveTaskAsset which defines parameters like the NPC type to spawn and the logic for determining its spawn location.

## Lifecycle & Ownership
The lifecycle of a BountyObjectiveTask is strictly controlled by its parent Objective and the server's objective management system.

-   **Creation:** An instance is created by the Objective framework when a player begins a quest containing this task. The `setup0` method is then invoked, which acts as the primary initializer. This is a critical, one-time operation that spawns the target entity and registers the task with the kill tracking system.
-   **Scope:** The object persists as long as the parent Objective is active for a player. Its state is serialized as part of the player's overall quest progression.
-   **Destruction:** The object is dereferenced and becomes eligible for garbage collection once its `complete` method is called and the parent Objective transitions its state. There is no explicit `destroy` method; cleanup, such as removing the map marker, is handled within the completion logic.

## Internal State & Concurrency
-   **State:** The BountyObjectiveTask is highly mutable. Its core state consists of the `entityUuid` of the spawned target and a `completed` boolean flag. This state is established during the `setup0` call and modified upon receiving a kill event. It does not cache any external data.
-   **Thread Safety:** **This class is not thread-safe.** All methods must be invoked from the main server game loop thread. Its operations on the entity `Store` and its internal state fields are not protected by locks. Unsynchronized access from other threads will lead to severe state corruption, race conditions, and server instability.

## API Surface
The public API is primarily intended for internal use by the objective system and event listeners.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(N) | Initializes the task. Spawns the target NPC, adds a map marker, and subscribes to the kill tracker. Complexity is tied to entity creation. |
| checkCompletion() | boolean | O(1) | Returns true if the target has been killed and the task is complete. |
| checkKilledEntity(store, npcRef, objective, npc, damageInfo) | void | O(1) | The core event callback. Checks if the killed entity matches the target UUID and, if so, triggers the task completion sequence. |
| toPacket(objective) | com.hypixel.hytale.protocol.ObjectiveTask | O(1) | Serializes the current task state into a network packet for client UI consumption. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly. Its behavior is defined by its corresponding asset file, and its lifecycle is managed by the engine. The engine's objective manager orchestrates its setup and relies on the event system for completion.

The conceptual flow managed by the engine is as follows:
```java
// Engine-level conceptual code
// 1. On objective activation, the engine calls setup0
TransactionRecord[] records = bountyTask.setup0(objective, world, store);
transactionManager.commit(records);

// 2. Later, when any entity is killed, the KillTrackerResource invokes the callback
// This happens automatically on a different call stack.
// killTracker.notifyListeners(killedEntityRef, damageInfo);
// -> which eventually calls:
// bountyTask.checkKilledEntity(store, killedEntityRef, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BountyObjectiveTask()`. The objective system is responsible for creating and managing task instances based on game assets.
-   **Manual State Mutation:** Do not set the `completed` field directly. The state must be driven by game events via the `checkKilledEntity` callback to ensure all cleanup logic (e.g., removing map markers, notifying the parent objective) is executed correctly.
-   **Re-running Setup:** Calling `setup0` more than once on the same instance will result in duplicate NPCs, broken state, and orphaned entities in the world.

## Data Pipeline
The BountyObjectiveTask participates in two primary data flows: initialization and completion.

> **Initialization Flow:**
> Objective Activation -> **BountyObjectiveTask.setup0** -> NPCPlugin -> EntityStore (NPC Spawned) -> MapMarker System (Marker Added) -> KillTrackerResource (Listener Registered)

> **Completion Flow:**
> Player Action (Kill) -> Damage System -> KillTrackerResource (Event Fired) -> **BountyObjectiveTask.checkKilledEntity** -> Internal State Update -> Objective.complete() -> MapMarker System (Marker Removed) -> Client UI Update
---

