---
description: Architectural reference for WorldEnvironmentSpawnData
---

# WorldEnvironmentSpawnData

**Package:** com.hypixel.hytale.server.spawning.world
**Type:** Transient State Object

## Definition
```java
// Signature
public class WorldEnvironmentSpawnData {
```

## Architecture & Concepts

The WorldEnvironmentSpawnData class is a stateful container that aggregates and manages all spawning-related data for a single, specific environment type within a world (e.g., a particular biome). It serves as the primary data source for the server's environment-based NPC spawning scheduler.

Its core responsibility is to maintain a real-time balance between the *desired* number of NPCs and the *actual* number of NPCs present. This is achieved by tracking two key metrics:

1.  **Expected NPCs:** A calculated target population based on the total area (segmentCount) of the environment currently loaded in memory and a configurable density value.
2.  **Actual NPCs:** A live counter of NPCs belonging to this environment that are currently spawned in the world.

The difference between these two values produces a "spawning pressure" metric, exposed via the spawnWeight method. A positive weight indicates the environment is underpopulated and requires more NPCs, driving the main spawning loop. This class does not perform the spawning itself; rather, it provides the necessary data and decision-making logic for a higher-level manager to execute spawn commands.

It maintains a set of references (Ref<ChunkStore>) to all loaded chunks that belong to its environment, allowing it to dynamically adjust the total spawnable area as players move through the world.

## Lifecycle & Ownership

-   **Creation:** An instance of WorldEnvironmentSpawnData is not created directly. It is instantiated and managed by a higher-level spawning service, likely when the first chunk corresponding to a new environment index is loaded into the world.
-   **Scope:** The object's lifetime is directly coupled to the presence of its associated environment in the active world state. It persists as long as at least one chunk of its environment type is loaded.
-   **Destruction:** When the last chunk belonging to this environment is unloaded, the managing service calls removeChunk. Once its internal chunkRefSet becomes empty, the managing service will release its reference to the instance, making it eligible for garbage collection.

## Internal State & Concurrency

-   **State:** This class is highly **mutable**. Its internal fields, such as segmentCount, actualNPCs, expectedNPCs, and the npcStatMap, are constantly updated in response to game events like chunk loading, entity spawning, and despawning. It acts as a live cache for the spawning state of an entire environment.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms or concurrent data structures. All methods perform non-atomic read-modify-write operations on shared state.

    **WARNING:** All interactions with an instance of WorldEnvironmentSpawnData MUST be synchronized externally. It is designed to be accessed exclusively from the main server thread to prevent race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addChunk(ref, accessor) | void | O(1) | Registers a chunk with this environment. Recalculates expected NPCs. |
| removeChunk(ref, accessor) | void | O(1) | Deregisters a chunk. Recalculates expected NPCs. |
| trackSpawn(roleIndex, count) | void | O(1) | Increments the actual NPC count for a given role. |
| trackDespawn(roleIndex, count) | void | O(1) | Decrements the actual NPC count for a given role. |
| spawnWeight() | double | O(1) | Returns the spawning "pressure". A positive value indicates a need for more NPCs. |
| pickRandomSpawnNPCStat(accessor) | WorldNPCSpawnStat | O(N) | Selects a weighted-random NPC type to spawn based on which roles are most underpopulated. N is the number of spawnable roles. |
| recalculateWeight(moonPhase) | void | O(N) | A more expensive operation to re-evaluate the spawn weights of all associated NPC roles, often triggered by game state changes like time of day. |

## Integration Patterns

### Standard Usage

This object is intended to be managed by a central spawning system. The manager holds a collection of WorldEnvironmentSpawnData instances, mapped by environment index. The typical interaction flow is event-driven.

```java
// Pseudocode for a hypothetical WorldSpawnManager
// On Chunk Load:
WorldEnvironmentSpawnData envData = getOrCreateEnvironmentData(chunk.getEnvironmentIndex());
envData.addChunk(chunk.getRef(), accessor);

// On NPC Spawn:
WorldEnvironmentSpawnData envData = getEnvironmentData(npc.getEnvironmentIndex());
envData.trackSpawn(npc.getRoleIndex(), 1);

// In the main spawning tick:
for (WorldEnvironmentSpawnData envData : allEnvironments) {
    if (envData.spawnWeight() > SPAWN_THRESHOLD) {
        WorldNPCSpawnStat statToSpawn = envData.pickRandomSpawnNPCStat(accessor);
        if (statToSpawn != null) {
            // ...initiate spawning logic for the selected NPC role
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new WorldEnvironmentSpawnData()`. The object will be unmanaged and will not receive updates from the game world, rendering its calculations useless. Always retrieve it from the authoritative spawning service.
-   **State Desynchronization:** The integrity of this object's state is entirely dependent on the caller. Forgetting to call `trackSpawn`/`trackDespawn` or `addChunk`/`removeChunk` will lead to a desynchronization between the object's internal state and the actual world state, causing severe bugs in the spawning system (e.g., overpopulation or empty worlds).
-   **Multi-threaded Access:** Accessing an instance from any thread other than the main server thread will cause unpredictable behavior, data corruption, and server crashes.

## Data Pipeline

WorldEnvironmentSpawnData acts as a central aggregation point for various asynchronous game events. It transforms these events into a coherent state that the spawning scheduler can query to make decisions.

> **Input Events:**
> Chunk Load/Unload -> `addChunk`/`removeChunk`
> NPC Spawn/Despawn -> `trackSpawn`/`trackDespawn`
> World Time Update -> `recalculateWeight`
>
> **State Aggregation:**
> **WorldEnvironmentSpawnData** (updates internal state: `segmentCount`, `actualNPCs`, `expectedNPCs`)
>
> **Output for Scheduler:**
> Spawning System Tick -> Queries `spawnWeight()` and `pickRandomSpawnNPCStat()` -> Spawning Decision & Execution

