---
description: Architectural reference for EntityDataSystem
---

# EntityDataSystem

**Package:** com.hypixel.hytale.component.system.data
**Type:** Framework Base Class

## Definition
```java
// Signature
public abstract class EntityDataSystem<ECS_TYPE, Q, R> extends ArchetypeDataSystem<ECS_TYPE, Q, R> {
```

## Architecture & Concepts

The EntityDataSystem is an abstract base class that forms the backbone of data processing within Hytale's Entity-Component-System (ECS) framework. It provides a high-level abstraction for iterating over and operating on entities that match a specific component layout, known as an Archetype.

Its primary architectural purpose is to decouple the *logic* of a system from the *execution strategy*. A developer implementing a concrete system (e.g., a PhysicsSystem) only needs to define the logic for processing a single entity. The EntityDataSystem base class, in conjunction with the engine's scheduler, handles the complex task of whether to execute this logic sequentially on the main thread or in parallel across multiple worker threads.

This is achieved through a dispatch mechanism in the static `doFetch` method. It inspects the result of the `isParallel` method and chooses one of two paths:
1.  **Sequential Path:** A simple loop that iterates through all entities in an `ArchetypeChunk` and calls the abstract `fetch` method for each one.
2.  **Parallel Path:** A more complex fork-join pattern. It breaks the `ArchetypeChunk` into ranges, wraps the processing logic in `SystemTaskData` objects, and submits them to the engine's task scheduler.

A critical concept in this design is the `CommandBuffer`. To prevent race conditions and maintain data integrity when modifying the ECS world structure (e.g., adding/removing components or entities), all such operations are deferred. In parallel execution, each worker thread receives a *forked* `CommandBuffer`. After all tasks are complete, these forked buffers are merged back into the main `CommandBuffer` in a deterministic manner.

## Lifecycle & Ownership

-   **Creation:** Concrete implementations of EntityDataSystem are not created manually. They are discovered and instantiated by the central ECS `SystemRegistry` or a similar manager during the engine's bootstrap sequence.
-   **Scope:** An instance of a system is a singleton for the lifetime of an ECS world. It persists from world creation to world destruction.
-   **Destruction:** The system instance is marked for garbage collection when the ECS world it belongs to is unloaded, for example, when a server shuts down or a client leaves a game session.

## Internal State & Concurrency

-   **State:** The EntityDataSystem base class is entirely stateless. Subclasses **must** also be designed to be stateless. Storing mutable state on a system instance is a severe anti-pattern that will break parallel execution and lead to race conditions. All per-entity state must be stored in Components.
-   **Thread Safety:** This class is designed to be inherently thread-safe through its architecture. The public API guarantees that when `isParallel` returns true, the abstract `fetch` method will be executed concurrently on different threads for different entities. The fork-join model for the `CommandBuffer` ensures that structural mutations to the ECS world are also handled in a thread-safe manner. The implementer of a concrete system is responsible for ensuring their `fetch` logic is re-entrant and does not access global mutable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isParallel() | boolean | O(1) | Hook for subclasses to declare their compatibility with parallel execution. Defaults to false. **CRITICAL:** Overriding this to true requires the `fetch` implementation to be fully re-entrant and thread-safe. |
| fetch(int, ArchetypeChunk, ...) | void | Varies | **Abstract.** The core logic hook that subclasses must implement. Processes a single entity at the given index within the chunk. |
| fetch(ArchetypeChunk, ...) | void | O(N) | The primary entry point called by the engine's scheduler. It processes an entire chunk of entities, delegating to the internal parallel/sequential dispatcher. |

## Integration Patterns

### Standard Usage

A developer extends EntityDataSystem, implements the abstract `fetch` method to contain the logic for a single entity, and optionally overrides `isParallel` to enable multi-threaded execution. The engine's scheduler is then responsible for invoking the system.

```java
// A concrete system for applying gravity to entities with a Velocity component.
// This system is simple and can be safely parallelized.
public class GravitySystem extends EntityDataSystem<MyEcsType, GravityQuery, Void> {

    @Override
    public boolean isParallel() {
        // This logic is safe to run on multiple threads.
        return true;
    }

    @Override
    public void fetch(int index, ArchetypeChunk<MyEcsType> chunk, Store<MyEcsType> store, CommandBuffer<MyEcsType> commandBuffer, GravityQuery query, List<Void> results) {
        // Get component data for the entity at the current index.
        VelocityComponent velocity = chunk.get(query.velocity, index);
        
        // Apply the system's logic.
        velocity.y -= 9.81f * Time.deltaTime;

        // Write the modified component data back.
        chunk.set(query.velocity, index, velocity);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new MySystem()`. Systems are managed by the ECS framework to guarantee their lifecycle and proper integration into the execution graph.
-   **Storing State:** Do not add member variables to your system subclass to store data that changes during execution. This will fail spectacularly in parallel mode. For example, do not maintain a `private int entitiesProcessedCount;` field.
-   **Direct World Modification:** Do not attempt to get a direct reference to the ECS world or `Store` to add or remove components from within the `fetch` method. All structural changes **must** go through the provided `CommandBuffer` instance.

## Data Pipeline

The flow of data and control for a single system execution pass is managed by the engine's scheduler and the `doFetch` dispatcher.

> Flow:
> Engine Scheduler -> **EntityDataSystem.fetch(chunk)** -> `doFetch` Dispatcher -> Parallel Task Splitter -> `SystemTaskData` worker -> **Subclass.fetch(index)** -> Forked CommandBuffer -> Post-run Merge Results & CommandBuffers

