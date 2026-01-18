---
description: Architectural reference for NPCBeaconSpawnJob
---

# NPCBeaconSpawnJob

**Package:** com.hypixel.hytale.server.spawning.jobs
**Type:** Transient / Pooled Object

## Definition
```java
// Signature
public class NPCBeaconSpawnJob extends SpawnJob {
```

## Architecture & Concepts

The NPCBeaconSpawnJob is a concrete implementation of the abstract SpawnJob, designed to handle the server-side spawning of Non-Player Characters (NPCs). It functions as a stateful, single-use command object within the server's entity spawning framework. Its primary role is to encapsulate all the parameters required to spawn a specific NPC or a group of them (a flock) in proximity to a player.

This class acts as a data carrier and a deferred factory resolver. It is initialized with high-level identifiers, such as a roleIndex and an optional FlockAsset. When the spawning system is ready to execute the job, it invokes getSpawnable, which resolves these identifiers into a concrete ISpawnableWithModel instance by querying the central NPCPlugin service. This decouples the spawn request logic from the NPC definition and creation logic, allowing for a flexible and data-driven spawning system.

The class name implies it is used in conjunction with a "beacon" system, a common pattern where in-world objects or zones trigger events, in this case, NPC spawn requests tied to a specific player.

### Lifecycle & Ownership
-   **Creation:** NPCBeaconSpawnJob instances are not meant to be instantiated directly using the new keyword. They are managed by a higher-level spawning service, which likely maintains a pool of job objects to reduce memory allocation overhead. A job is acquired from this pool when a spawn is requested.
-   **Scope:** The object is active for the duration of a single spawning operation. Its lifecycle begins when `beginProbing` is called to configure it and ends when the spawning system either completes the spawn, aborts it via `shouldTerminate`, or explicitly finishes with the job.
-   **Destruction:** The object is not typically destroyed by the garbage collector. Instead, the `reset` method is called to clear its internal state, after which it is returned to the object pool to be reused for a future spawn request. The `shouldTerminate` method provides a vital signal to the owning system that the job's context (specifically the target player) is no longer valid, allowing for early termination.

## Internal State & Concurrency
-   **State:** This class is highly mutable. Its fields (`player`, `roleIndex`, `flockAsset`, etc.) are populated by the `beginProbing` method and represent the complete state of a single spawn request. The state is transient and is invalidated upon calling `reset`.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and manipulated by a single thread at a time, typically within the server's main tick loop or a dedicated spawning thread. Concurrent calls to `beginProbing` or modification of its state from multiple threads will lead to race conditions and unpredictable behavior. All synchronization must be handled externally by the managing spawner system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| beginProbing(targetPlayer, spawns, roleIndex, flockDef) | void | O(1) | Initializes the job with all required parameters for a new spawn request. This is the primary entry point for configuring the job. |
| getSpawnable() | ISpawnableWithModel | O(1) | Resolves the configured `roleIndex` into a concrete spawnable NPC definition. Throws `IllegalArgumentException` if the role is invalid, abstract, or does not implement the required interface. |
| shouldTerminate() | boolean | O(1) | Returns true if the job should be aborted. This is primarily used to check if the target player reference has become invalid (e.g., the player disconnected). |
| budgetAvailable() | boolean | O(1) | Indicates if the job can be executed based on system resources. **Warning:** This implementation always returns true, implying that beacon-triggered spawns bypass standard spawning budgets and are considered high-priority. |
| reset() | void | O(1) | Clears all internal state, preparing the object to be returned to a pool for reuse. |

## Integration Patterns

### Standard Usage

The intended use involves a spawner service acquiring a job from a pool, configuring it, and submitting it to an execution queue. The executor then uses the job to create the entity.

```java
// Conceptual example within a SpawnerService
NPCBeaconSpawnJob job = jobPool.acquire(NPCBeaconSpawnJob.class);

// Configure the job with data from a game event
job.beginProbing(player, 1, someRoleIndex, someFlockAsset);

// Submit to the execution system
spawnScheduler.submit(job);

// The scheduler will later call job.getSpawnable() to get the entity model
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new NPCBeaconSpawnJob()`. This bypasses the object pooling mechanism, leading to unnecessary garbage collection and performance degradation. Always acquire jobs from the designated manager or pool.
-   **State Corruption:** Do not modify the job's state after it has been submitted to a scheduler. The job should be treated as immutable by the caller after `beginProbing` is invoked.
-   **Premature Access:** Do not call `getSpawnable` before `beginProbing` has been successfully executed. The internal state will be uninitialized, leading to exceptions or incorrect behavior.

## Data Pipeline

The flow of data for a beacon-triggered NPC spawn involves several distinct systems, with NPCBeaconSpawnJob acting as the data container that moves between them.

> Flow:
> Game Event (Player enters beacon area) -> SpawnerService -> **NPCBeaconSpawnJob** (Acquired & Configured) -> SpawnScheduler -> EntityFactory (uses `getSpawnable`) -> EntityStore (NPC created in world)

