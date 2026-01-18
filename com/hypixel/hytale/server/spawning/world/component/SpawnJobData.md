---
description: Architectural reference for SpawnJobData
---

# SpawnJobData

**Package:** com.hypixel.hytale.server.spawning.world.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class SpawnJobData implements Component<ChunkStore> {
```

## Architecture & Concepts
The SpawnJobData class is a stateful data container that encapsulates the context and progress of a single, discrete entity spawning operation on the server. It is not a service or manager; rather, it functions as a "scratchpad" or "context object" for the server's spawning pipeline.

This component is designed to be attached to a ChunkStore, scoping its data to the processing of a specific world chunk. It aggregates all necessary parameters for a spawn attempt, including:
*   **What to spawn:** Defined by SpawnWrapper, FlockAsset, and roleIndex.
*   **Environmental context:** The Environment in which the spawn is attempted.
*   **Execution state:** Tracks metrics like totalColumnsTested, budgetUsed, and success/failure counts.
*   **Outcome analysis:** The rejectionMap provides detailed reasons for spawn failures, which is critical for server-side tuning and debugging.

By centralizing this state, the core spawning algorithm can remain stateless, operating purely on the data provided by a given SpawnJobData instance. This makes the spawning logic easier to reason about and test.

## Lifecycle & Ownership
The lifecycle of a SpawnJobData object is intentionally short and managed to minimize performance overhead, particularly garbage collection.

-   **Creation:** These objects are not intended for direct instantiation. The presence of a parameter-heavy `init` method strongly implies an object pooling pattern managed by the core SpawningPlugin. A spawner service retrieves a recycled object from a pool and calls `init` to configure it for a new job.

-   **Scope:** The object's lifetime is ephemeral, scoped exclusively to a single spawning evaluation within a single server tick. It exists only for the duration required to determine if a spawn can occur at a specific location.

-   **Destruction:** The object is not destroyed in the traditional sense. After a job is completed or aborted, the `terminate` method is called, and the instance is expected to be returned to its object pool. Its state is considered invalid until `init` is called again for a subsequent spawning job.

## Internal State & Concurrency
-   **State:** SpawnJobData is highly mutable by design. Its primary function is to be modified by the spawning algorithm as it progresses through various checks and stages. It contains numerous counters, flags, and references that are updated throughout the evaluation.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms such as locks or atomic variables. The static `jobIdCounter` is not atomic, indicating that the entire spawning system is designed to operate on a single thread.

    **WARNING:** Accessing or modifying a SpawnJobData instance from any thread other than the main server thread for that world will lead to race conditions, data corruption, and unpredictable server behavior. All interactions must be synchronized with the server's main tick loop.

## API Surface
The public API is designed for state management by the spawning system. Most methods are simple accessors or mutators. The key operational methods are listed below.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(...) | void | O(1) | Resets and configures the component for a new spawn job. This is the primary entry point for reusing a pooled object. |
| terminate() | void | O(1) | Marks the job as complete, signaling that it can be returned to an object pool. |
| adjustBudgetUsed(int) | void | O(1) | Modifies the budget consumed by the job. Central to the spawning system's performance management. |
| incrementTotalColumnsTested() | void | O(1) | Increments a performance counter. Called repeatedly during the spawn evaluation phase. |
| getSpawningContext() | SpawningContext | O(1) | Provides access to the nested context object containing detailed world state for the spawn attempt. |
| getRejectionMap() | Object2IntMap | O(1) | Returns the map used for tracking spawn failure reasons. |

## Integration Patterns

### Standard Usage
A developer should never interact with this class directly. It is an internal component managed by the server's spawning engine. The conceptual flow is as follows.

```java
// Conceptual example of engine usage
// Engine acquires a pooled SpawnJobData instance for a ChunkStore
SpawnJobData jobData = chunkStore.getOrAddComponent(SpawnJobData.getComponentType());

// Engine configures the job for spawning a wolf pack
jobData.init(roleIndex, environment, envIndex, wolfSpawnWrapper, wolfFlockAsset, 5);

// Engine passes the job to the processing pipeline
spawningPipeline.processJob(jobData);

// After processing, the job is terminated
jobData.terminate();
// The component is now ready to be reused in a future tick
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SpawnJobData()`. The system relies on a managed lifecycle, likely through object pooling, to prevent excessive GC pressure. Always acquire instances through the appropriate engine or component system API.

-   **State Persistence:** Do not hold references to a SpawnJobData instance across multiple server ticks. Its data is only valid for the immediate spawning operation.

-   **Re-use Without Initialization:** Using a terminated SpawnJobData object without first calling `init` will cause the new spawn job to execute with stale data from a previous operation, resulting in highly unpredictable and incorrect behavior.

-   **Cloning:** The `clone` method explicitly throws UnsupportedOperationException. These objects are unique state machines for a single job and are not meant to be copied.

## Data Pipeline
SpawnJobData acts as the payload that flows through the internal spawning pipeline. It collects data and state changes at each step.

> Flow:
> Spawning Engine Trigger -> Object Pool provides **SpawnJobData** -> `init` populates with spawn parameters -> Location Finding Algorithm (updates `totalColumnsTested`) -> Condition Evaluation (updates `rejectionMap` and `budgetUsed`) -> Spawn Decision -> Entity Creation (if successful) -> **SpawnJobData** is terminated and returned to pool.

