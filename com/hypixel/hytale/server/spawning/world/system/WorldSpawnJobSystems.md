---
description: Architectural reference for WorldSpawnJobSystems
---

# WorldSpawnJobSystems

**Package:** com.hypixel.hytale.server.spawning.world.system
**Type:** Utility

## Definition
```java
// Signature
public class WorldSpawnJobSystems {
    // Inner classes define ECS Systems
    public static class Ticking extends EntityTickingSystem<ChunkStore> { ... }
    public static class EntityRemoved extends HolderSystem<ChunkStore> { ... }
    public static class TickingState extends RefChangeSystem<ChunkStore, NonTicking<ChunkStore>> { ... }
}
```

## Architecture & Concepts

WorldSpawnJobSystems is a stateless utility class that encapsulates the core logic for executing an individual NPC spawn attempt within a specific world chunk. It is not a persistent service but rather a collection of static methods and, more importantly, the Entity Component System (ECS) definitions that drive the server-side world population process.

This class acts as the "worker" in the world spawning pipeline. Its primary responsibility is to process a **SpawnJobData** component attached to a **WorldChunk** entity. The job involves an iterative, budget-constrained search for a valid spawn location within the chunk's columns. Upon finding a suitable location, it orchestrates the creation of the NPC entity via the NPCPlugin.

The logic is partitioned into three inner classes, each representing a distinct ECS system that manages a part of the spawn job's lifecycle:

*   **Ticking:** The primary execution system. It runs every tick on chunks with an active SpawnJobData component, performing the search and spawn logic.
*   **EntityRemoved:** A cleanup system. It ensures that if a chunk entity is destroyed while a spawn job is pending, the job is properly terminated and its statistics are recorded.
*   **TickingState:** A state-change cleanup system. It handles the case where a chunk becomes inactive (e.g., unloaded or too far from players) by terminating any associated spawn job.

This design decouples the *request* for a spawn (the SpawnJobData component) from the *execution* of the spawn, allowing the engine to manage spawning as a background, asynchronous-style task that is resilient to world state changes.

### Lifecycle & Ownership

The WorldSpawnJobSystems class itself is a stateless utility and is never instantiated. The lifecycle described here pertains to the spawn *process* managed by its inner ECS systems, which is tied directly to the **SpawnJobData** component.

*   **Creation:** A spawn job begins when an external system, typically the higher-level WorldSpawnSystem, adds a SpawnJobData component to a WorldChunk entity. This action effectively queues the chunk for a spawn attempt.
*   **Scope:** The job is considered active as long as the SpawnJobData component exists on a ticking WorldChunk entity. The Ticking system will process the job on each server tick. A single job may persist across multiple ticks if its initial budget is exhausted before a result is determined, in which case it will resume on the next tick.
*   **Destruction:** A spawn job is terminated under one of the following conditions:
    1.  **Success:** An NPC is successfully spawned. The Ticking system removes the SpawnJobData component via a CommandBuffer.
    2.  **Failure:** The system exhausts all possible spawn locations or determines the NPC type is unspawnable in the chunk. The Ticking system removes the component.
    3.  **Abortion (Non-Ticking):** The chunk becomes non-ticking. The TickingState system detects this state change and terminates the job, removing the component.
    4.  **Abortion (Entity Removal):** The chunk entity itself is removed from the world. The EntityRemoved system intercepts this event and terminates the job.

## Internal State & Concurrency

*   **State:** WorldSpawnJobSystems is fundamentally stateless. All operational state is stored externally in components, primarily **SpawnJobData**, or in world-level resources like **WorldSpawnData**. The process is highly stateful, but the class itself is not.
*   **Thread Safety:** The core spawning logic is **not thread-safe**. The Ticking system explicitly declares `isParallel` as false, forcing the ECS scheduler to execute all spawn jobs serially within a single thread. This is critical to prevent race conditions when iterating chunk columns, modifying spawn statistics, and creating entities. Access to shared resources like SpawnSuppressionController is handled carefully, using concurrent data structures where necessary (e.g., Long2ObjectConcurrentHashMap).

**WARNING:** Do not attempt to parallelize the Ticking system. The underlying logic in SpawningContext and the iterative search process are not designed for concurrent execution and will lead to data corruption or unpredictable behavior.

## API Surface

The public contract of this class consists of its inner ECS system definitions, which are intended for registration with the server's ECS world. Direct invocation of its static methods is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Ticking | class | O(N) | The primary ECS system that executes spawn jobs. Complexity is proportional to the number of active jobs. |
| EntityRemoved | class | O(1) | An ECS system that handles cleanup when a chunk with a spawn job is removed. |
| TickingState | class | O(1) | An ECS system that handles cleanup when a chunk with a spawn job becomes non-ticking. |

## Integration Patterns

### Standard Usage

The systems defined within WorldSpawnJobSystems must be instantiated and registered with the appropriate ECS Store during server initialization. The engine handles the rest of the lifecycle. The system is driven entirely by the presence of components.

```java
// During server or world initialization
// (Conceptual example)
ChunkStore chunkStore = world.getChunkStore();

// Register the systems that drive the spawn job process
chunkStore.addSystem(new WorldSpawnJobSystems.Ticking(...));
chunkStore.addSystem(new WorldSpawnJobSystems.EntityRemoved(...));
chunkStore.addSystem(new WorldSpawnJobSystems.TickingState(...));

// To trigger a spawn job, another system adds the component to a chunk entity
Ref<ChunkStore> chunkRef = ...;
CommandBuffer<ChunkStore> cmd = ...;
cmd.addComponent(chunkRef, new SpawnJobData(...));
```

### Anti-Patterns (Do NOT do this)

*   **Direct Method Invocation:** Never call the private static methods like `run` or `trySpawn` directly. They are internal implementation details of the Ticking system and rely on the specific context provided by the ECS loop.
*   **Component Mismanagement:** Do not manually remove a SpawnJobData component without going through the proper termination logic in `endProbing`. Doing so will fail to update world spawn statistics, leading to an inaccurate census of active jobs and spawned entities.
*   **Concurrent Modification:** Do not modify a SpawnJobData component from another system while it is being processed. The Ticking system is not parallel and assumes exclusive ownership during its tick execution.

## Data Pipeline

The data flow for a single spawn job is a linear process orchestrated by the Ticking system.

> Flow:
> **WorldSpawnSystem** (or other manager) adds **SpawnJobData** component to a **WorldChunk** entity.
> -> **WorldSpawnJobSystems.Ticking** system queries for entities with SpawnJobData.
> -> For each job, the system reads chunk block data, **ChunkEnvironmentSpawnData**, and world-level **SpawnSuppressionController** data.
> -> It iterates through potential spawn columns within the chunk, testing for valid placement against geometry, lighting, and breathability rules.
> -> **Success:** Calls **NPCPlugin.spawnEntity**, which creates a new **NPCEntity** in the **EntityStore**. Updates **WorldSpawnData** statistics. Removes the **SpawnJobData** component.
> -> **Failure:** Updates **WorldSpawnData** statistics with failure reasons. Removes the **SpawnJobData** component.

