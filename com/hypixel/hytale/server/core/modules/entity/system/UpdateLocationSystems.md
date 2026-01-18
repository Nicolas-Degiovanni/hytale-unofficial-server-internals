---
description: Architectural reference for UpdateLocationSystems
---

# UpdateLocationSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition
```java
// Signature
public class UpdateLocationSystems {

    // System for handling initial entity placement
    public static class SpawnSystem extends RefSystem<EntityStore> {
        // ...
    }

    // System for handling continuous entity movement
    public static class TickingSystem extends EntityTickingSystem<EntityStore> {
        // ...
    }
}
```

## Architecture & Concepts

UpdateLocationSystems is a critical server-side utility that serves as the bridge between an entity's physical position in the world and its logical grouping within a chunk. It is not a single system but a container for two distinct, highly-focused Entity-Component-Systems (ECS): a SpawnSystem and a TickingSystem. Together, these systems ensure that every entity with a TransformComponent is correctly associated with a WorldChunk.

The core responsibility of this module is to monitor an entity's position and, upon detecting a chunk boundary crossing, execute the complex logic required to transfer the entity's reference from its old chunk to its new one.

Key architectural features include:

*   **Separation of Concerns:** The logic is split into two phases. The SpawnSystem handles the *initial* placement of an entity into a chunk when it first enters the world. The TickingSystem handles *continuous* updates every tick for entities that move.
*   **Asynchronous Chunk Loading:** The system is designed to be non-blocking. If an entity moves into a chunk that is not currently loaded in memory, the system initiates an asynchronous request to the ChunkStore. The entity's final chunk transition is processed in a callback once the chunk data is available, preventing the main server thread from stalling on disk I/O.
*   **State Reconciliation:** It is the sole authority for modifying the chunk reference within an entity's TransformComponent and updating the entity lists within a chunk's EntityChunk component. This centralized control prevents state desynchronization between an entity's perceived location and its actual container.
*   **Safety and Recovery:** The module contains robust fallback mechanisms. Non-player entities moving into invalid or unloadable chunks are safely removed from the world. Players are teleported to a nearby safe location to prevent them from getting stuck or falling through the world.

### Lifecycle & Ownership
-   **Creation:** The nested SpawnSystem and TickingSystem classes are discovered, instantiated, and registered by the server's central SystemRegistry during world initialization. The parent UpdateLocationSystems class is a static utility container and is never instantiated.
-   **Scope:** The registered system instances persist for the entire lifetime of the World they are associated with. They are fundamental to world operation.
-   **Destruction:** The systems are deregistered and marked for garbage collection when the World is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** UpdateLocationSystems is entirely stateless. It acts as a pure function container, operating exclusively on the state held within the components passed to it, primarily TransformComponent, WorldChunk, and EntityChunk. All state changes are performed through a CommandBuffer or directly on the component stores.

-   **Thread Safety:** This system is **not thread-safe** for direct, external invocation. Its methods are designed to be executed exclusively by the main server world thread. The internal asynchronous chunk loading path is carefully managed; the final state mutation is marshaled back onto the main world thread via the world's scheduler to guarantee safe component access.

    **WARNING:** Never call the internal static methods like updateLocation from an external thread. Doing so will bypass the engine's command buffer and thread-safety mechanisms, leading to race conditions and world corruption.

## API Surface

The public API consists of the two nested ECS classes, which are automatically managed by the engine. Direct interaction is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SpawnSystem | class | O(log N) | An ECS system that runs once when an entity with a TransformComponent is created. Complexity depends on whether the target chunk is loaded. |
| TickingSystem | class | O(1) to O(log N) | An ECS system that runs every tick for all entities with a TransformComponent. Complexity is O(1) if the entity remains in its chunk, but higher if a chunk transition triggers an async load. |

## Integration Patterns

### Standard Usage

A developer does not interact with these systems directly. The engine's SystemRegistry automatically executes them. To have an entity managed by UpdateLocationSystems, simply add a TransformComponent to it. The systems will then handle its location management automatically.

```java
// Correct integration is implicit.
// By adding a TransformComponent, the entity is automatically
// processed by UpdateLocationSystems.TickingSystem every tick.

// Example: Spawning an entity
Holder<EntityStore> entityHolder = EntityStore.REGISTRY.newHolder();
entityHolder.addComponent(TransformComponent.getComponentType(), new TransformComponent(initialPosition));

// The CommandBuffer adds the entity to the world, which triggers
// UpdateLocationSystems.SpawnSystem to run.
commandBuffer.addEntity(entityHolder, AddReason.SPAWN);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Do not call `UpdateLocationSystems.updateLocation` or any other static helper. These are internal implementation details and lack the necessary execution context (like a valid CommandBuffer) provided by the ECS runner.
-   **Manual Chunk Reference Management:** Do not manually set the chunk reference on a TransformComponent. This will corrupt the world state, as the corresponding EntityChunk will not be updated, leading to the entity becoming "lost" to chunk-based queries and systems.

## Data Pipeline

The flow of data and logic for an entity changing chunks is a primary function of this module. The pipeline differs depending on whether the destination chunk is already in memory.

**Scenario 1: Entity moves into a loaded chunk (Synchronous Path)**
> Flow:
> Game Tick -> **TickingSystem.tick()** -> Read TransformComponent position -> Detect chunk boundary cross -> **updateLocation()** -> Get new chunk reference from ChunkStore (immediate) -> **updateChunk()** -> Update old/new EntityChunk components -> Update TransformComponent chunk reference

**Scenario 2: Entity moves into an unloaded chunk (Asynchronous Path)**
> Flow:
> Game Tick -> **TickingSystem.tick()** -> Read TransformComponent position -> Detect chunk boundary cross -> **updateLocation()** -> Request chunk from ChunkStore (async) -> *[Tick processing for this entity ends]*
>
> *[Later, on world thread pool]*
>
> ChunkStore loads chunk -> CompletableFuture completes -> **updateChunkAsync()** -> **updateChunk()** -> Update old/new EntityChunk components -> Update TransformComponent chunk reference

