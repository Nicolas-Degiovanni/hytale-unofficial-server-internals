---
description: Architectural reference for KillSpawnBeaconObjectiveTask
---

# KillSpawnBeaconObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.task
**Type:** Transient

## Definition
```java
// Signature
public class KillSpawnBeaconObjectiveTask extends KillObjectiveTask {
```

## Architecture & Concepts

The KillSpawnBeaconObjectiveTask is a specialized implementation within the server-side Adventure Mode objective system. It represents a concrete, stateful task that requires a player to eliminate entities spawned from one or more designated spawn beacons. It extends the more generic KillObjectiveTask, inheriting the core logic for tracking kill counts, but adds a critical setup phase.

This class acts as the bridge between the abstract objective definition (the Asset) and the concrete game world state. Its primary responsibility is to translate a configuration into live, interactive game elements. During its setup phase, it dynamically spawns `LegacySpawnBeaconEntity` instances into the world at locations relative to the parent Objective's position.

This task integrates with several core server systems:
*   **Objective System:** It is managed by a parent Objective, which controls its lifecycle.
*   **Spawning System:** It directly invokes the SpawningPlugin to create beacon entities from predefined spawn assets.
*   **World & Entity System:** It queries the world for valid spawn locations and creates new entities within the world's EntityStore.
*   **Transactional State Management:** It uses a system of TransactionRecords to log the world-mutating actions it performs, such as spawning entities. This ensures that the objective system can track, and potentially roll back, changes made to the world.
*   **Event-Driven Progress:** After setup, it registers a KillTaskTransaction with the global KillTrackerResource. This follows an Observer pattern, allowing the task to be passively notified of relevant kill events without actively polling game state.

## Lifecycle & Ownership

*   **Creation:** An instance of KillSpawnBeaconObjectiveTask is not created directly. It is instantiated by the objective framework's deserialization mechanism via its static CODEC field. This typically occurs when a parent Objective is activated and its list of tasks is processed from an asset file.
*   **Scope:** The object's lifetime is tightly coupled to its parent Objective. It persists as long as the objective is active in the game world. If the parent Objective is completed, failed, or unloaded, this task instance is dereferenced.
*   **Destruction:** The object is eligible for garbage collection once its parent Objective is no longer active and the KillTrackerResource releases its reference to the associated KillTaskTransaction. There is no explicit `destroy` method; cleanup relies on the Java garbage collector.

## Internal State & Concurrency

*   **State:** This class is **mutable** and stateful. It caches the TransactionRecords generated during its initial setup in the `serializedTransactionRecords` field (inherited). This prevents the expensive operation of re-calculating spawn locations and re-creating beacon entities if the setup logic is invoked more than once.
*   **Thread Safety:** This class is **not thread-safe**. All interactions with this object, particularly the `setup0` method, must occur on the main server thread. Its operations involve direct mutation of world state (via ComponentAccessor and World), which are not designed for concurrent access. Access from other threads will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsset() | KillSpawnBeaconObjectiveTaskAsset | O(1) | Returns the immutable asset that defines this task's configuration. |
| setup0(objective, world, store) | TransactionRecord[] | O(N) | **Core Logic.** Spawns N configured beacons into the world. Registers a listener for kill events. This method is idempotent due to internal caching of transaction records. Throws exceptions on critical failures. |

## Integration Patterns

### Standard Usage

This class is an internal component of the objective system and is not intended for direct use by most developers. A game designer or scripter would define the task's behavior declaratively in an asset file, which the engine then uses to create and manage an instance.

A conceptual asset definition might look like this:

```json
// In an objective asset file
{
  "taskType": "Hytale:KillSpawnBeacon",
  "spawnBeacons": [
    {
      "spawnBeaconId": "goblin_camp_beacon_1",
      "offset": { "x": 10, "y": 0, "z": 5 }
    },
    {
      "spawnBeaconId": "goblin_camp_beacon_2",
      "offset": { "x": -10, "y": 0, "z": 5 }
    }
  ],
  // ... other properties from KillObjectiveTaskAsset
}
```

The engine's ObjectivePlugin would then be responsible for instantiating and managing the KillSpawnBeaconObjectiveTask based on this data.

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Never call `new KillSpawnBeaconObjectiveTask()`. The object is deeply tied to the objective framework's lifecycle and state management, which is handled via the CODEC deserializer. Manual creation will result in a non-functional, isolated object.
*   **Manual Invocation of setup0:** Do not call the `setup0` method outside of the objective lifecycle management system. The framework ensures it is called at the correct time and with the correct world context. Calling it manually can lead to duplicate entities or state desynchronization.
*   **Multi-threaded Access:** As stated in the concurrency section, never access an instance of this class from any thread other than the main server world thread.

## Data Pipeline

The flow of control and data for this task is divided into two distinct phases: Setup and Monitoring.

> **Setup Flow:**
> Objective Asset Definition -> ObjectivePlugin Deserialization -> **KillSpawnBeaconObjectiveTask** instance created -> `setup0` invoked -> SpawningPlugin -> `LegacySpawnBeaconEntity.create` -> EntityStore Mutation -> TransactionRecord returned

> **Monitoring Flow:**
> Player Kills Entity -> Server Kill Event -> KillTrackerResource -> Notifies `KillTaskTransaction` -> **KillSpawnBeaconObjectiveTask** state is updated -> Objective progress is advanced

