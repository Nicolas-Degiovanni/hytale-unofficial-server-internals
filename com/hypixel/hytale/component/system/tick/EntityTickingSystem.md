---
description: Architectural reference for EntityTickingSystem
---

# EntityTickingSystem

**Package:** com.hypixel.hytale.component.system.tick
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class EntityTickingSystem<ECS_TYPE> extends ArchetypeTickingSystem<ECS_TYPE> {
```

## Architecture & Concepts

The EntityTickingSystem is a foundational abstract class within the Entity-Component-System (ECS) framework. It serves as a high-performance template for creating game logic systems that operate on entities sharing a common set of components, known as an Archetype.

This class acts as the primary bridge between the engine's main game loop and the per-entity update logic. Its core architectural purpose is to abstract the complex task of iterating over entities and to provide a seamless, opt-in mechanism for parallel execution. By handling the orchestration of serial versus parallel processing, it allows developers of concrete systems (e.g., a PhysicsSystem or AISystem) to focus solely on the logic for a single entity.

The system's design heavily relies on the **Command Buffer** pattern to ensure thread safety during parallel execution. Instead of mutating component data directly, parallel tasks write their intended changes to a thread-local CommandBuffer. These buffers are then merged back into the main world state at a safe synchronization point, preventing data races and ensuring deterministic outcomes.

## Lifecycle & Ownership

-   **Creation:** Concrete implementations of EntityTickingSystem are not created dynamically during gameplay. They are instantiated once by the core engine, typically via a System Registry or dependency injection framework, during the world-loading or application bootstrap phase.
-   **Scope:** An instance of a system persists for the entire lifetime of the game world. It is a long-lived, stateless service object.
-   **Destruction:** The system is destroyed and garbage collected only when the game world is unloaded or the application terminates.

## Internal State & Concurrency

-   **State:** This base class is inherently stateless. All necessary context (delta time, entity data, command buffers) is passed as arguments to the tick methods. Concrete subclasses are strongly encouraged to remain stateless to facilitate safe parallel execution. Any state stored in a subclass must be managed with extreme care to avoid concurrency issues.
-   **Thread Safety:** The class is designed for conditional thread safety. The main entry point, tick, is expected to be called from a single, main game thread. The class then internally manages dispatching work to a thread pool if parallelism is enabled. The critical mechanism for safety is the use of forked CommandBuffer instances for each worker thread, which isolates all mutations. Direct, concurrent calls to a system instance from multiple external threads are unsupported and will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isParallel(chunkSize, taskCount) | boolean | O(1) | Hook for subclasses to enable parallel processing for a given chunk. The default implementation returns false. |
| tick(dt, archetypeChunk, store, commandBuffer) | void | O(N) | The primary entry point called by the engine. Orchestrates the iteration over all entities in the chunk. |
| tick(dt, index, archetypeChunk, store, commandBuffer) | void | O(1) | **Abstract Method.** The core logic to be implemented by subclasses. Processes a single entity at the given index. |
| doTick(system, dt, chunk, store, buffer) | static void | O(N) | The internal static worker that contains the branching logic for serial vs. parallel execution. |
| SystemTaskData.invokeParallelTask(task, buffer) | static void | O(M) | Static utility to execute all scheduled parallel tasks and merge their forked CommandBuffers. M is the number of parallel tasks. |

## Integration Patterns

### Standard Usage

A developer extends EntityTickingSystem to implement specific game logic. The core responsibility is to implement the abstract tick method that processes a single entity. To unlock performance gains on multi-core processors, the developer can override isParallel.

```java
// A concrete system for updating entity positions
public class MovementSystem extends EntityTickingSystem<MyEcsType> {

    // Enable parallel execution if there are more than 1024 entities to process
    @Override
    public boolean isParallel(int archetypeChunkSize, int taskCount) {
        return archetypeChunkSize > 1024;
    }

    // Implement the logic for a single entity
    @Override
    public void tick(float dt, int index, @Nonnull ArchetypeChunk<MyEcsType> chunk, @Nonnull Store<MyEcsType> store, @Nonnull CommandBuffer<MyEcsType> commandBuffer) {
        // Safely get component data for the entity at 'index'
        Position pos = chunk.get(index, Position.class);
        Velocity vel = chunk.get(index, Velocity.class);

        // Calculate the new position
        float newX = pos.x + vel.dx * dt;
        float newY = pos.y + vel.dy * dt;

        // Do NOT modify 'pos' directly. Instead, queue a command to set the new state.
        // This is critical for thread safety.
        commandBuffer.setComponent(chunk.getEntity(index), new Position(newX, newY));
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new MovementSystem()`. Systems are managed by the engine's lifecycle and should be retrieved from a central registry or context. Direct instantiation bypasses engine management.
-   **Direct State Mutation:** Never modify component data directly within the tick method (e.g., `pos.x = newX;`). This will cause severe data corruption and race conditions when parallel execution is enabled. Always use the provided CommandBuffer to queue changes.
-   **Blocking Operations:** Avoid performing blocking operations like file I/O or network requests inside the tick method. This will stall the processing thread and severely degrade engine performance, especially in a parallel context.

## Data Pipeline

The EntityTickingSystem directs the flow of entity processing through one of two paths based on the result of the isParallel check.

### Serial Execution Path
> Flow:
> Game Loop -> **EntityTickingSystem.tick()** -> doTick() -> Simple `for` loop iterates from 0 to N -> `subclass.tick(index)` -> Writes to main CommandBuffer

### Parallel Execution Path
> Flow:
> Game Loop -> **EntityTickingSystem.tick()** -> doTick() -> Creates ParallelRangeTask -> Forks CommandBuffer for each task -> Task Scheduler dispatches `SystemTaskData.accept(index)` to worker threads -> `subclass.tick(index)` writes to forked CommandBuffer -> **SystemTaskData.invokeParallelTask()** -> Merges all forked CommandBuffers into the main CommandBuffer

