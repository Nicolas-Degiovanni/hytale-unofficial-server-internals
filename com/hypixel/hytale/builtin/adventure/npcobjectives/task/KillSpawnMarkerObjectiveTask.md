---
description: Architectural reference for KillSpawnMarkerObjectiveTask
---

# KillSpawnMarkerObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.task
**Type:** Stateful Task Object

## Definition
```java
// Signature
public class KillSpawnMarkerObjectiveTask extends KillObjectiveTask {
```

## Architecture & Concepts
The KillSpawnMarkerObjectiveTask is a specialized implementation of a "kill" objective task within Hytale's adventure mode system. It extends the base KillObjectiveTask, inheriting its core responsibility of tracking entity kills, but adds a critical preliminary step: activating nearby entity spawners.

This class acts as a bridge between the high-level Objective system and the server's Spawning system. Its primary architectural function is to initiate a dynamic combat encounter when an objective begins. Instead of tracking kills of pre-existing entities, it first queries the world's spatial partitioning data (via SpatialResource) to find specific spawn markers within a radius defined in its corresponding asset. It then triggers these markers, causing them to spawn the entities that the player must subsequently defeat.

The behavior is entirely data-driven by a KillSpawnMarkerObjectiveTaskAsset, which specifies the target spawn marker identifiers and the search radius. The actual tracking of kills is delegated to the parent class logic, which is managed through a KillTaskTransaction registered with the global KillTrackerResource.

### Lifecycle & Ownership
-   **Creation:** An instance is created by the Objective system when an objective containing this task is activated. The static CODEC field indicates that it is typically deserialized from game data assets, not instantiated manually.
-   **Scope:** The object's lifetime is strictly bound to its parent Objective. It exists only while the objective is active.
-   **Destruction:** The instance is eligible for garbage collection once the parent Objective is completed, failed, or otherwise unloaded. The KillTaskTransaction it creates is unregistered from the KillTrackerResource, breaking the reference cycle.

## Internal State & Concurrency
-   **State:** The class is largely stateless beyond its initial configuration, which is stored in its asset and is immutable after creation. It does not cache world data or track kill counts directly. All dynamic state, such as the number of entities killed, is managed externally by the KillTrackerResource and its associated TransactionRecord.
-   **Thread Safety:** This class is **not thread-safe** and must be manipulated only from the main server world thread. The setup0 method performs several operations that are thread-sensitive:
    -   It uses a thread-local list from SpatialResource, confining the spatial query to the calling thread.
    -   Crucially, it schedules the actual spawn marker triggering via `world.execute()`. This lambda is queued for execution on the main world tick, ensuring that entity state modifications are performed safely and without race conditions.

    **WARNING:** Calling methods on this class from an asynchronous task or a different thread will lead to concurrency violations and world state corruption.

## API Surface
The primary interaction point is the protected `setup0` method, called by the objective management system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsset() | KillSpawnMarkerObjectiveTaskAsset | O(1) | Retrieves the immutable asset that configures this task. |
| setup0(objective, world, store) | TransactionRecord[] | O(N) | Activates the task. Queries for and triggers spawn markers within the configured radius. N is the number of entities in the spatial query. Creates and registers a transaction for kill tracking. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is instantiated and managed by the engine's objective system based on asset definitions. A content creator would define the task in a data file, and the engine handles the lifecycle.

The conceptual flow within the engine is as follows:
```java
// Engine-level code (conceptual)
// An objective is activated for a player
Objective objective = player.getQuestState().getActiveObjective();
World world = player.getWorld();
Store<EntityStore> store = world.getStore();

// The engine iterates through the objective's tasks
for (ObjectiveTask task : objective.getTasks()) {
    if (task instanceof KillSpawnMarkerObjectiveTask) {
        // The setup method is called, triggering spawners and starting the kill tracking
        TransactionRecord[] records = task.setup0(objective, world, store);
        objective.addTransactionRecords(records);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new KillSpawnMarkerObjectiveTask()`. The object is tightly coupled to its asset and the objective lifecycle, and must be created by the engine's deserialization and objective management systems.
-   **Multiple Setups:** Calling `setup0` more than once for the same task instance will result in re-triggering spawn markers and registering duplicate, redundant kill tracking transactions. This can lead to incorrect objective logic and performance degradation.
-   **External State Modification:** Do not attempt to modify the task's state after it has been initialized. Its behavior is defined entirely by its asset.

## Data Pipeline
The data flow for this task occurs in two distinct phases: setup and tracking.

> **Setup Flow:**
> Objective Activation -> Engine calls `KillSpawnMarkerObjectiveTask.setup0` -> Queries `SpatialResource` using objective position and asset radius -> Filters results by spawn marker IDs from asset -> Schedules `SpawnMarkerEntity.trigger()` on World thread -> Creates `KillTaskTransaction` -> Registers transaction with `KillTrackerResource`

> **Tracking Flow (Handled by external systems):**
> Entity Death Event -> `KillTrackerResource` -> Notifies `KillTaskTransaction` -> `Objective` progress is updated -> Objective completion check is triggered

