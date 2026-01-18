---
description: Architectural reference for SpawnJobSystem
---

# SpawnJobSystem

**Package:** com.hypixel.hytale.server.spawning.controllers
**Type:** System Component

## Definition
```java
// Signature
public abstract class SpawnJobSystem<J extends SpawnJob, T extends SpawnController<J>> extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts

The SpawnJobSystem is an abstract base class that forms the core of the server's NPC and entity spawning logic. It operates within the server's Entity Component System (ECS) framework as a "System" responsible for executing stateful, budget-aware spawning operations.

Architecturally, this class employs the **Template Method** design pattern. It defines the high-level, invariant algorithm for processing a queue of spawn requests within the `tickSpawnJobs` and `runJob` methods. This core loop handles job selection, budget management, and state transitions. Concrete subclasses are required to implement the variable parts of the algorithm, such as the logic for picking a valid spawn location (`pickSpawnPosition`) and the specific conditions for a successful spawn (`trySpawn`).

This system is fundamentally **budget-aware**. It is designed to perform a limited amount of work per server tick, defined by the `blockBudget`. This ensures that intensive spawning operations, such as searching for valid locations in the world, do not cause server performance degradation or frame rate drops. The system will process jobs until its budget for the current tick is exhausted, resuming on subsequent ticks if necessary.

It acts as the *executor* in a paired relationship with a `SpawnController`, which acts as the *manager*. The `SpawnController` maintains the lists of active and idle `SpawnJob` objects, while the SpawnJobSystem is responsible for processing the active jobs from that controller.

## Lifecycle & Ownership

-   **Creation:** A concrete implementation of SpawnJobSystem is instantiated by the server's ECS engine during world initialization. Systems are typically registered with the engine and created automatically, not manually.
-   **Scope:** Session-scoped. An instance of a SpawnJobSystem persists for the entire lifetime of the world it is associated with.
-   **Destruction:** The system is destroyed and cleaned up when the world is unloaded or the server shuts down. It does not own the `SpawnJob` objects it processes; ownership of jobs is maintained by the corresponding `SpawnController`.

## Internal State & Concurrency

-   **State:** The SpawnJobSystem is entirely **stateless**. It holds no mutable instance fields and relies solely on the state passed into its methods, primarily the `SpawnController` and the individual `SpawnJob` objects. All state related to a spawning operation is encapsulated within the `SpawnJob` itself.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. As an `EntityTickingSystem`, it is designed to be invoked synchronously once per game tick. All world-mutating operations (e.g., creating an entity) **must** be deferred through the provided `CommandBuffer`. This is a critical pattern to prevent race conditions and ensure deterministic updates to the world state, which is processed at the end of the tick.

## API Surface

The primary API consists of the `protected` methods intended for use and implementation by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tickSpawnJobs(controller, store, buffer) | void | O(N * M) | The main entry point for subclasses. Iterates through active jobs in the controller, executing them until the per-tick budget is exhausted. N is the number of active jobs; M is the work performed per job. |
| onStartRun(spawnJob) | void | O(1) | A hook method called at the beginning of `runJob`. Resets the budget used for the job. |
| onEndProbing(...) | void | *Abstract* | A hook for subclasses to perform cleanup or logging after a job's location search (probing) is complete. |
| pickSpawnPosition(...) | boolean | *Abstract* | **Template Method.** Subclasses must implement the logic to find and validate a potential spawn location. |
| trySpawn(...) | Result | *Abstract* | **Template Method.** Subclasses must implement the logic to attempt the spawn at the position found by `pickSpawnPosition`. |
| spawn(...) | Result | *Abstract* | **Template Method.** Subclasses must implement the final entity creation logic, typically by issuing commands to the `CommandBuffer`. |

## Integration Patterns

### Standard Usage

A concrete class must extend SpawnJobSystem and implement the abstract methods. This new system is then registered with the server's ECS engine to be ticked automatically.

```java
// A concrete implementation for hostile mob spawning
public class MobSpawnJobSystem extends SpawnJobSystem<MobSpawnJob, MobSpawnController> {

    // The system is ticked by the engine, which calls this method.
    @Override
    public void tick(TickContext<EntityStore> context, Store<EntityStore> store, CommandBuffer<EntityStore> buffer) {
        MobSpawnController controller = getMobSpawnController(store);
        if (controller != null) {
            // Defer to the base class's core processing loop.
            tickSpawnJobs(controller, store, buffer);
        }
    }

    @Override
    protected boolean pickSpawnPosition(MobSpawnController controller, MobSpawnJob job, CommandBuffer<EntityStore> buffer) {
        // Custom logic to find a dark, valid location for a mob.
        // ...
        return locationFound;
    }

    @Override
    protected SpawnJobSystem.Result trySpawn(MobSpawnController controller, MobSpawnJob job, CommandBuffer<EntityStore> buffer) {
        // Custom logic to check player distance, pack size, etc.
        // If conditions are met, call spawn().
        // ...
        return spawn(job.getWorld(), controller, job, buffer);
    }
    
    // Other abstract method implementations...
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new MobSpawnJobSystem()`. Systems are managed by the ECS engine's lifecycle. Manually creating an instance will result in a non-functional system that is never ticked.
-   **Concurrent Access:** Never call `tickSpawnJobs` or other methods from an asynchronous task or a different thread. This will lead to severe concurrency issues, including `ConcurrentModificationException` and world state corruption.
-   **Direct World Modification:** Implementations of `spawn` or other methods must **never** modify the `World` or `EntityStore` directly. All entity creation, component addition, or other state changes must be queued via the provided `CommandBuffer`.

## Data Pipeline

The flow of data and control for a single spawning operation is orchestrated by the system in conjunction with its controller and the broader ECS engine.

> Flow:
> SpawningPlugin (requests a spawn) -> SpawnController (creates and enqueues a `SpawnJob`) -> **SpawnJobSystem.tickSpawnJobs** (selects the job from the controller) -> **runJob** (executes the job within budget) -> Subclass `pickSpawnPosition` / `trySpawn` (implements specific logic) -> Subclass `spawn` (issues an entity creation command) -> CommandBuffer (queues the command) -> ECS Engine (executes the command at end of tick) -> World State Updated

