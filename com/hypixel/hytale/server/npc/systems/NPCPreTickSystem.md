---
description: Architectural reference for NPCPreTickSystem
---

# NPCPreTickSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class NPCPreTickSystem extends SteppableTickingSystem {
```

## Architecture & Concepts
The NPCPreTickSystem is a foundational component of the server-side Entity Component System (ECS) responsible for managing the lifecycle of Non-Player Characters (NPCs). As a "PreTick" system, it is scheduled to execute early in the main game loop for each server tick.

Its primary architectural responsibilities are:
1.  **State Caching:** It captures and stores the initial position of each NPC at the very beginning of a tick. This provides a stable, predictable reference point for other game logic systems, such as AI or physics, that run later in the same tick.
2.  **Despawn Management:** It implements the complete state machine for NPC despawning. This includes initiating despawn timers, triggering despawn animations, and ultimately commanding the removal of the NPC entity from the world. This centralizes despawn logic, preventing scattered and inconsistent implementations across different NPC types.
3.  **System Ordering:** It explicitly declares its execution order relative to other systems. By specifying that it must run *before* DeathSystems.CorpseRemoval, it guarantees that despawning logic is fully resolved before any corpse-handling logic begins, preventing race conditions and ordering paradoxes.

The system operates on a specific subset of entities defined by an ECS Query, processing only those entities that possess both an NPCEntity component and a TransformComponent.

## Lifecycle & Ownership
-   **Creation:** The NPCPreTickSystem is instantiated automatically by the server's core ECS framework during world initialization. It is not intended for manual creation by game logic developers.
-   **Scope:** An instance of this system persists for the entire lifetime of a game world. Its lifecycle is bound directly to the server session.
-   **Destruction:** The system is destroyed and its resources are reclaimed only when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This system is **stateless**. It holds no mutable fields that persist between ticks or across entities. All state related to its operation, such as despawn timers or animation flags, is stored directly within the NPCEntity components of the entities it processes. This design is crucial for enabling safe parallel execution.
-   **Thread Safety:** The system is designed to be **thread-safe** and can be executed in parallel by the ECS scheduler. Safety is achieved through two core ECS patterns:
    1.  **Data Partitioning:** The ECS scheduler feeds the system disjoint sets of entities (ArchetypeChunks), ensuring that no two threads can write to the same component data simultaneously.
    2.  **Deferred Structural Changes:** All world-altering operations, such as entity removal, are not performed immediately. Instead, they are enqueued as commands onto a CommandBuffer. These commands are then flushed and executed at a designated synchronization point later in the tick, preventing structural modifications during parallel processing.

## API Surface
The public methods of this class are part of the contract with the ECS scheduler and are not intended for direct invocation by game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDependencies() | Set | O(1) | Returns the set of execution order dependencies. |
| isParallel(int, int) | boolean | O(1) | Determines if the system can be run in parallel for a given workload. |
| getQuery() | Query | O(1) | Returns the ECS query that defines which entities this system will process. |
| steppedTick(...) | void | O(1) per entity | The core logic executed by the scheduler for each entity matching the query. |

## Integration Patterns

### Standard Usage
Developers do not call this system directly. Integration is implicit: any server-side entity created with both an **NPCEntity** component and a **TransformComponent** will be automatically discovered and managed by this system every tick.

The primary interaction is configuring the NPCEntity component's properties, which this system will then read to perform its logic.

```java
// Example: Creating an entity that will be managed by NPCPreTickSystem
// This code would exist within a spawner or game logic system.

CommandBuffer commands = ...;
EntityStore store = ...;

// The presence of NPCEntity and TransformComponent makes the entity
// eligible for processing by NPCPreTickSystem.
commands.createEntity(
    new NPCEntity(someRole, someConfig),
    new TransformComponent(initialPosition)
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new NPCPreTickSystem()`. The resulting object would not be registered with the ECS scheduler and would have no effect on the game world.
-   **Manual Invocation:** Do not call the `steppedTick` method directly. Doing so bypasses the ECS scheduler, dependency ordering, and the CommandBuffer system, which will lead to severe concurrency issues, data corruption, and unpredictable crashes.
-   **Stateful Logic:** Do not modify this system to hold state in its own fields. All state must be stored in components to maintain thread safety and adhere to ECS principles.

## Data Pipeline
The NPCPreTickSystem acts as a processor within the main server tick. It reads component data, modifies it, and outputs commands for later execution.

> Flow:
> ECS Scheduler identifies eligible entities -> **NPCPreTickSystem.steppedTick** is invoked for each entity -> Reads `TransformComponent` and `NPCEntity` -> Updates internal state of `NPCEntity` (e.g., timers) -> Writes `removeEntity` or `run` commands to the `CommandBuffer` -> ECS Synchronization Point executes commands from the buffer.

