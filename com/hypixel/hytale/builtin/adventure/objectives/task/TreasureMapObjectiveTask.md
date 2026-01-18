---
description: Architectural reference for TreasureMapObjectiveTask
---

# TreasureMapObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Stateful Task Instance

## Definition
```java
// Signature
public class TreasureMapObjectiveTask extends ObjectiveTask {
```

## Architecture & Concepts
The TreasureMapObjectiveTask is a concrete implementation of the ObjectiveTask contract, designed to manage quest objectives that require players to locate and interact with one or more hidden treasure chests. It operates entirely on the server-side, acting as the state machine and world-interaction controller for this specific type of quest.

Its core responsibilities include:
1.  **World Modification:** Spawning specially-configured treasure chests in the game world at procedurally determined locations based on configuration from a TreasureMapObjectiveTaskAsset.
2.  **State Management:** Tracking which chests have been spawned, their unique identifiers (UUIDs), and how many have been opened by players. This state is serializable via its static CODEC, allowing quest progress to persist across server restarts.
3.  **Event Handling:** Subscribing to the server's event bus to listen for TreasureChestOpeningEvent. This event-driven approach decouples the task from the chest block's direct implementation.
4.  **Player Feedback:** Creating and removing map markers to guide players to chest locations and sending network packets to update the client-side UI with the current progress.
5.  **Transactional Integrity:** All world modifications, such as spawning a chest, are wrapped in a TransactionRecord. This ensures that actions are logged and auditable, which is critical for system stability and debugging.

This class is a central component of the adventure mode objective system, bridging the gap between high-level quest configuration (Assets) and low-level world state and player interaction (BlockStates, Events).

### Lifecycle & Ownership
-   **Creation:** An instance of TreasureMapObjectiveTask is created by the parent Objective during its setup phase. This process is typically triggered when a player accepts a quest. The class is either instantiated directly with its corresponding Asset configuration or deserialized from persistent storage if the objective was already in progress.
-   **Scope:** The object's lifetime is strictly bound to its parent Objective. It persists as long as the objective is active for any player.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the parent Objective is completed, abandoned, or otherwise removed from the game state. There is no explicit destruction method; cleanup relies on the JVM's garbage collector.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains a running count of opened chests (*currentCompletion*), the total number of chests required (*chestCount*), and a list of unique identifiers for each spawned chest (*chestUUIDs*). This state is fundamental to its operation and is serialized for persistence.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed exclusively from the main server thread that manages the corresponding World instance. All interactions, including event handling and state modification, must occur on this thread to prevent race conditions, world corruption, and inconsistent state.

    **WARNING:** Asynchronous or multi-threaded access to an instance of this class will lead to undefined behavior and server instability.

## API Surface
The public API is minimal, as the class is primarily driven by the Objective system and internal event handlers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(Objective, World, Store) | TransactionRecord[] | O(N * M) | Initializes the task. Spawns N chests, each taking up to M attempts. Registers event listeners. **CRITICAL:** This is the primary entry point and must be called before the task can be processed. |
| checkCompletion() | boolean | O(1) | Returns true if the number of opened chests meets the required count. |
| toPacket(Objective) | com.hypixel.hytale.protocol.ObjectiveTask | O(1) | Serializes the current task state into a network packet for client-side UI updates. |
| getAsset() | TreasureMapObjectiveTaskAsset | O(1) | Provides typed access to the underlying configuration asset. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is instantiated and managed by the core objective system. A designer or developer would configure a TreasureMapObjectiveTaskAsset, and the engine handles the lifecycle automatically.

```java
// Conceptual example of how the engine uses this class.
// DO NOT replicate this pattern.

// 1. Objective is initialized from an asset
TreasureMapObjectiveTaskAsset taskAsset = loadAsset("my_treasure_quest.json");
Objective parentObjective = getActiveObjective();
ObjectiveTask task = new TreasureMapObjectiveTask(taskAsset, 0, 0);

// 2. The objective system calls setup0
TransactionRecord[] records = task.setup0(parentObjective, world, entityStore);
transactionManager.commit(records);

// 3. Later, an event handler is invoked by the event bus
// This happens automatically when a player opens a chest.
// TreasureChestOpeningEvent event = ...;
// task.onTreasureChestOpeningEvent(parentObjective, event);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new TreasureMapObjectiveTask()`. The object will be in an invalid state, lacking its asset configuration and proper registration within the parent Objective. This will result in NullPointerExceptions and failed objective tracking.
-   **Manual State Mutation:** Do not modify the `currentCompletion` or `chestUUIDs` fields directly. All state changes must flow through the `onTreasureChestOpeningEvent` handler to ensure game logic, network updates, and persistence are correctly triggered.
-   **Re-using Instances:** An instance of this class is tied to a single Objective instance. Do not attempt to reuse it for a different objective, as its internal state will be incorrect.

## Data Pipeline
The flow of data and control for this task follows two distinct paths: setup and progress tracking.

**Setup Flow:**
> Objective Asset -> `Objective.setup()` -> **TreasureMapObjectiveTask.setup0()** -> `calculateChestSpawnPosition()` -> `spawnChestBlock()` -> World State Change (Block placed) & `TransactionRecord` -> Map Marker Added

**Progress Tracking Flow:**
> Player opens chest -> `TreasureChestState` logic -> `TreasureChestOpeningEvent` published -> Server Event Bus -> **TreasureMapObjectiveTask.onTreasureChestOpeningEvent()** -> Internal state updated -> `Objective.markDirty()` -> `toPacket()` -> Network Packet sent to Client -> Client UI updates

