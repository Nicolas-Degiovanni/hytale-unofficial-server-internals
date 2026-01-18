---
description: Architectural reference for SpawnJob
---

# SpawnJob

**Package:** com.hypixel.hytale.server.spawning.jobs
**Type:** Transient Base Class

## Definition
```java
// Signature
public abstract class SpawnJob {
```

## Architecture & Concepts

The SpawnJob class is an abstract base class that represents a single, stateful unit of work within the server's entity spawning system. It is the foundational component for defining and executing a specific spawning attempt, such as placing a creature in the world.

This class is not intended for direct instantiation. Instead, it provides a contract and shared machinery for concrete implementations (e.g., a theoretical MonsterSpawnJob). Its primary role is to encapsulate the entire process of evaluating spawn conditions, managing resource consumption, and ultimately producing a spawnable entity.

Key architectural concepts include:

-   **Unit of Work:** Each SpawnJob instance corresponds to one discrete spawning operation. It is created, executed, and then discarded.
-   **Budgeting System:** The class incorporates a simple budgeting mechanism (columnBudget, budgetUsed) to enforce performance constraints. This ensures that a single, complex spawning evaluation does not consume excessive server resources in a single tick, preventing performance degradation.
-   **State Encapsulation:** The job holds all necessary state for its operation within its instance fields and the associated SpawningContext object. This context is progressively populated as the job probes the world for valid spawn locations.
-   **Template Method Pattern:** SpawnJob uses the template method pattern by defining the overall execution skeleton while deferring specific implementation details—such as what to spawn (getSpawnable) and when to stop (shouldTerminate)—to its subclasses.

## Lifecycle & Ownership

A SpawnJob's lifecycle is intentionally brief and managed by the core spawning service. Understanding this lifecycle is critical to avoid memory leaks or invalid state.

-   **Creation:** A concrete SpawnJob is instantiated by a higher-level manager, such as a SpawningPlugin or a world chunk spawner, when a spawning opportunity arises. It is never created directly by general game logic. The constructor assigns a unique, sequential jobId.
-   **Scope:** The object's lifetime is tied to a single spawning evaluation, which may last for one or more server ticks if the job is complex and budget-constrained. It does not persist beyond this evaluation.
-   **Destruction:** The SpawnJob is eligible for garbage collection as soon as the managing system completes or abandons the job and drops its reference. There are no explicit cleanup methods; its resources, particularly the SpawningContext, are expected to be released via the reset method before the object is discarded.

## Internal State & Concurrency

-   **State:** A SpawnJob is highly mutable. Its internal fields, such as budgetUsed and terminated, are modified throughout its execution. The nested SpawningContext object also accumulates state as the job probes the environment.
-   **Thread Safety:** **This class is not thread-safe and must not be shared across threads.**
    -   The static jobIdCounter is incremented in the constructor without any synchronization. Concurrent instantiation from multiple threads will result in a race condition, potentially assigning duplicate job IDs and leading to unpredictable behavior.
    -   All instance fields are accessed without locks. The design assumes a single-threaded owner, such as the main server thread, which will create, execute, and discard the job.

## API Surface

The public and protected API defines the contract for both the managing system and concrete subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| beginProbing() | void | O(1) | **Lifecycle Method.** Prepares the job for execution by resetting its context and termination flag. Must be called before the job is processed. |
| budgetAvailable() | boolean | O(1) | Checks if the job has remaining budget to continue its work. |
| isTerminated() | boolean | O(1) | Returns true if the job has been flagged for termination, either by itself or an external system. |
| getSpawnable() | ISpawnableWithModel | Varies | **Abstract.** Subclasses must implement this to return the final entity model to be spawned. Returns null if no suitable spawn was found. |
| shouldTerminate() | boolean | Varies | **Abstract.** Subclasses must implement this to define the conditions under which the job considers its work complete. |
| getSpawningContext() | SpawningContext | O(1) | Provides access to the context object where environmental data is stored during probing. |

## Integration Patterns

### Standard Usage

A SpawnJob is orchestrated by a managing service. The typical pattern involves creating a job, configuring its budget, and executing it within a processing loop until it is terminated or runs out of budget.

```java
// A hypothetical SpawningManager executing a job
ConcreteSpawnJob job = new ConcreteSpawnJob();
job.setColumnBudget(100); // Set resource limit
job.beginProbing();

// In the main server tick loop...
while (!job.isTerminated() && job.budgetAvailable()) {
    // The job performs a small piece of work internally,
    // using its budget and updating its context.
    job.doWorkTick(); 
}

if (job.getSpawnable() != null) {
    world.spawnEntity(job.getSpawnable());
}
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Creation:** Do not create SpawnJob instances from multiple threads. The static jobIdCounter is not thread-safe and will cause critical failures. All job creation must be serialized through a single thread or a synchronized factory.
-   **State Re-use without Reset:** Do not attempt to re-run a completed job without first calling beginProbing. Failure to do so will result in the job executing with stale context and termination flags.
-   **External Context Mutation:** Avoid modifying the job's SpawningContext from an external system. The context is owned and managed exclusively by the job during its execution.

## Data Pipeline

The SpawnJob acts as the central processing and decision-making stage in the entity spawning pipeline.

> Flow:
> Spawning Trigger (e.g., Chunk Load) -> Spawning Service -> **SpawnJob Instantiation & Configuration** -> Job Execution (World Probing & Context Population) -> **getSpawnable()** -> Entity Factory -> World Insertion

