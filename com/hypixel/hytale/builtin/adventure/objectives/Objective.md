---
description: Architectural reference for Objective
---

# Objective

**Package:** com.hypixel.hytale.builtin.adventure.objectives
**Type:** Transient State Model

## Definition
```java
// Signature
public class Objective implements NetworkSerializable<com.hypixel.hytale.protocol.Objective> {
```

## Architecture & Concepts

The Objective class is the central, stateful representation of a single, active quest instance within the game world. It is not the definition of a quest; rather, it is a live object instantiated from a static **ObjectiveAsset** template. Each Objective instance tracks the real-time progress for a specific set of players.

Architecturally, an Objective functions as a state machine. Its lifecycle is defined by a sequence of **TaskSets**, where each TaskSet represents a distinct stage or phase of the quest. The Objective manages an array of **ObjectiveTask** objects, which are the concrete, checkable conditions that players must satisfy to complete the current TaskSet (e.g., "kill 5 goblins", "talk to NPC X").

A critical design feature is its integration with a transactional system for world modification. When an ObjectiveTask is set up via the **setup** method, it may need to spawn entities, create markers, or alter the game world. These actions are recorded as **TransactionRecords**. This ensures that if any part of the setup fails, or if the objective is cancelled, all associated world changes can be safely and atomically rolled back, preventing corrupted game state.

The Objective is designed for persistence and network synchronization. It implements the **NetworkSerializable** interface, allowing its state to be converted into a network packet for client-side UI updates. Furthermore, its static **CODEC** field enables robust serialization and deserialization, allowing active quests to be saved and reloaded across server sessions.

## Lifecycle & Ownership

-   **Creation:** An Objective is never instantiated directly. It is created and managed exclusively by the **ObjectivePlugin**, which acts as a central registry and factory. An instance is typically spawned when a player or group triggers a quest-starting event, using a specific ObjectiveAsset as its configuration template.

-   **Scope:** The object's lifetime is tied directly to the quest's activity. It persists as long as the quest is active for the assigned players. Due to its serializable nature, its state can survive server restarts, being reloaded by the ObjectivePlugin during world bootstrap.

-   **Destruction:** An Objective is terminated under two conditions. Upon successful completion via the **complete** method, it executes final reward logic and is marked as complete. The ObjectivePlugin will then remove it from active tracking. Alternatively, if the quest is abandoned or fails, the **cancel** method is invoked to revert all transactional world changes before the object is deregistered and garbage collected. The **unload** method serves a similar cleanup purpose during server shutdown.

## Internal State & Concurrency

-   **State:** The Objective class is highly mutable, as its primary function is to track dynamic quest progress. Key mutable fields include **currentTaskSetIndex**, the array of **currentTasks**, and the **completed** flag. It maintains a **dirty** flag to optimize persistence, ensuring that only changed objectives are written to storage.

-   **Thread Safety:** This class is **not thread-safe** and is designed to be manipulated exclusively on its parent **World**'s main thread. State transition methods like **setup**, **checkTaskSetCompletion**, and **complete** perform world modifications and must be wrapped in a `world.execute(...)` call to prevent race conditions and world corruption.
    -   **Warning:** The single exception is the **activePlayerUUIDs** set, which is a thread-safe collection. This allows player login or logout events, which may occur on network threads, to safely update the list of currently online participants without blocking the main game loop. All other collections and fields are unsynchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup(Store) | boolean | O(N) | Initializes the tasks for the current TaskSet. Returns false on failure. **Must be called on the world thread.** |
| checkTaskSetCompletion(Store) | boolean | O(N) | Checks if all current tasks are complete. If so, triggers progression to the next TaskSet or completes the objective. |
| complete(Store) | void | O(C) | Finalizes the objective, runs completion handlers, and notifies players. C is the number of completion handlers. |
| cancel() | void | O(N) | Aborts the objective and reverts all transactional world changes made by its tasks. |
| reloadObjectiveAsset(Map) | void | O(N) | Asynchronously reloads the objective's tasks from a new asset definition, enabling live quest updates. |
| toPacket() | Objective | O(N) | Serializes the current objective state into a network packet for client synchronization. |

*N = number of tasks in the current TaskSet.*

## Integration Patterns

### Standard Usage

An Objective instance is always managed by a higher-level system, typically the **ObjectivePlugin**. The plugin is responsible for creating, persisting, and driving state changes in response to game events.

```java
// Pseudo-code demonstrating management by a plugin
Objective objective = objectivePlugin.startObjectiveForPlayer(player, "tutorial_quest_1");

// Setup the initial tasks
world.execute(() -> {
    objective.setup(world.getEntityStore().getStore());
});

// Later, in response to a game event...
world.execute(() -> {
    // This call will advance the objective state if tasks are complete
    objective.checkTaskSetCompletion(world.getEntityStore().getStore());
    objective.markDirty(); // Flag for persistence
});
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new Objective()`. This bypasses the ObjectivePlugin's tracking and persistence mechanisms, resulting in an unmanaged and ephemeral quest that will not save or load correctly.

-   **Cross-Thread Modification:** Do not call methods like **setup**, **complete**, or **checkTaskSetCompletion** from any thread other than the objective's parent world thread. Doing so will lead to severe concurrency issues, data corruption, and server instability.

-   **State Tampering:** Do not manually modify the **currentTasks** array or the **currentTaskSetIndex**. All state transitions must be driven through the public API, such as **checkTaskSetCompletion**, to ensure transactional integrity and proper event handling.

## Data Pipeline

The Objective class is a key component in two primary data flows: server-side state progression and client-side UI updates.

> **Server-Side State Progression Flow:**
> Game Event (e.g., EntityDeathEvent) -> ObjectiveTask Listener -> Task marked complete -> **Objective.checkTaskSetCompletion()** -> State transition (TaskSet advances or Objective completes) -> World state is modified

> **Client-Side Synchronization Flow:**
> Objective state change -> **Objective.toPacket()** -> TrackOrUpdateObjective packet created -> Network Layer -> Client receives packet -> Client-side Objective UI is updated

