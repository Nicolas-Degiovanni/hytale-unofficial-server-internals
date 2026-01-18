---
description: Architectural reference for DespawnSystem
---

# DespawnSystem

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** System Component

## Definition
```java
// Signature
public class DespawnSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The DespawnSystem is a server-side system within the Hytale Entity Component System (ECS) framework. Its sole responsibility is to manage the lifecycle of entities that are marked for automatic removal after a specific time. It embodies the "System" part of ECS, containing pure logic that operates on a filtered set of entities.

This system is a critical component for world maintenance and performance optimization. It prevents the accumulation of temporary entities, such as item drops, temporary visual effects, or short-lived creatures, which would otherwise persist indefinitely and consume server resources.

Architecturally, it hooks into the main server game loop via its base class, EntityTickingSystem. The ECS scheduler identifies this system and executes its *tick* method once per game loop iteration. The system uses a highly efficient, pre-computed **Query** to select only the entities that possess a **DespawnComponent** and, crucially, do **not** have an **Interactable** component. This exclusion prevents the system from removing an entity while a player might be interacting with it, a key design choice for gameplay stability.

### Lifecycle & Ownership
- **Creation:** The DespawnSystem is instantiated once by the server's ECS world during its initialization phase. The system is registered with the world's scheduler, which then manages its execution. Developers do not create instances of this class directly.
- **Scope:** The system's lifetime is bound to the server's world instance. It persists as long as the world is loaded and running.
- **Destruction:** The instance is garbage collected when the corresponding server world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The DespawnSystem is effectively stateless. Its internal fields, *despawnComponentType* and *query*, are final references configured at construction. It does not store or cache any per-entity data. All state it operates on is external, read from entity components and global resources like TimeResource.

- **Thread Safety:** This system is designed for parallel execution. The *isParallel* method signals to the ECS scheduler that the *tick* method can be safely invoked concurrently on different blocks of entities (ArchetypeChunks). Thread safety is guaranteed by the ECS architecture itself:
    1. Each parallel task operates on a disjoint set of entities.
    2. All modifications, such as entity removal, are not performed immediately. Instead, they are recorded as commands in a thread-local **CommandBuffer**.
    3. These command buffers are synchronized and executed at a later, safe point in the game loop.

**WARNING:** Manually invoking the *tick* method from an unmanaged thread will bypass these safety guarantees and lead to severe concurrency issues, including data corruption and crashes.

## API Surface
The public API is exclusively for consumption by the ECS scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the pre-compiled query used to select entities for processing. |
| isParallel(archetypeChunkSize, taskCount) | boolean | O(1) | A heuristic that informs the scheduler whether to execute this system's logic in parallel. |
| tick(dt, index, archetypeChunk, store, commandBuffer) | void | O(1) | The core logic executed for each entity matching the query. Checks the despawn time and enqueues a removal command if necessary. |

## Integration Patterns

### Standard Usage
A developer never interacts with the DespawnSystem directly. To schedule an entity for despawning, one simply adds a **DespawnComponent** to it. The system will automatically discover and manage the entity.

```java
// Example: Making an entity despawn 30 seconds from now.
// This code would typically be in another system or entity creation logic.

TimeResource time = world.getResource(TimeResource.class);
Instant despawnTime = time.getNow().plusSeconds(30);

// The CommandBuffer is used to safely add the component.
commandBuffer.addComponent(myEntityRef, new DespawnComponent(despawnTime));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DespawnSystem()`. The ECS framework is responsible for creating and managing system instances. Direct instantiation will result in a non-functional system that is not registered with the game loop.
- **Manual Invocation:** Never call the *tick* method directly. This bypasses the scheduler, thread management, and the CommandBuffer synchronization mechanism, which is critical for maintaining a consistent world state.
- **Ignoring the Interactable Rule:** Adding a DespawnComponent to an entity that also has an Interactable component will have no effect. The system's query is designed to explicitly ignore such entities to prevent gameplay conflicts.

## Data Pipeline
The system's data flow is simple and unidirectional, transforming temporal state into a lifecycle command.

> Flow:
> ECS Scheduler invokes tick -> **DespawnSystem** reads TimeResource -> **DespawnSystem** reads DespawnComponent from entity -> Time is compared -> If expired, a removeEntity command is written to the CommandBuffer -> The scheduler later executes the CommandBuffer, removing the entity from the world.

