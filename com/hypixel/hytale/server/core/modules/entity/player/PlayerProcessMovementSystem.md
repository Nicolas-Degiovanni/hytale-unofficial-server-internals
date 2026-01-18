---
description: Architectural reference for PlayerProcessMovementSystem
---

# PlayerProcessMovementSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System Service

## Definition
```java
// Signature
public class PlayerProcessMovementSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerProcessMovementSystem is a core server-side system within the Entity Component System (ECS) framework responsible for processing player movement each tick. It acts as the primary authority for translating a player's intended velocity into a final, collision-aware position within the world.

This system is not a general-purpose physics engine. Its scope is strictly limited to entities that possess a specific set of components, as defined by its internal query: Player, BoundingBox, Velocity, CollisionResultComponent, and PositionDataComponent. This design ensures it only operates on player-controlled entities that are actively participating in the world simulation.

Architecturally, this class serves as an orchestrator. It reads the current state from an entity's components, delegates the complex and computationally expensive task of collision detection to the global CollisionModule, and then applies the results back to the entity. It also handles the gameplay consequences of movement, such as triggering block interactions or applying environmental damage, by scheduling deferred commands on a CommandBuffer.

## Lifecycle & Ownership
- **Creation:** A single instance of PlayerProcessMovementSystem is instantiated by the server's central System Registry during the server bootstrap sequence. It is configured at creation time with the specific ComponentType instances it will query for.
- **Scope:** Session-scoped. The system persists for the entire lifetime of the server instance. It is designed to be stateless with respect to individual players; all per-entity state is read from and written to components during the tick method.
- **Destruction:** The instance is discarded and eligible for garbage collection only during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is effectively immutable after construction. Its internal fields are final references to component types and a pre-built query object. It does not cache or store any per-entity state between ticks. All operational state is passed into the tick method via the ArchetypeChunk and Store parameters.

- **Thread Safety:** **This system is not thread-safe and must be executed serially.** The implementation of the isParallel method explicitly returns false, which serves as a contract with the ECS scheduler. This contract guarantees that the tick method will never be invoked concurrently for different entities. This is a critical safety measure to prevent race conditions when accessing shared, non-thread-safe resources like the World, ChunkStore, and the global CollisionModule.

    **WARNING:** Modifying this system to run in parallel without a fundamental re-architecture of the underlying world and collision modules will lead to severe data corruption and server instability.

## API Surface
The public API is minimal, designed for integration with the ECS scheduler, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the pre-built ECS query. Used by the scheduler to identify which entities this system should process. |
| isParallel(...) | boolean | O(1) | Returns false. Informs the scheduler that this system's tick method cannot be executed in parallel. |
| tick(...) | void | O(N) | Executes one iteration of the movement logic for a single entity. Complexity is dominated by the call to CollisionModule.findIntersections, where N is the number of potential colliders in the entity's vicinity. |

## Integration Patterns

### Standard Usage
Direct interaction with this system is not a standard development pattern. Developers enable this system's logic to run on an entity by ensuring the entity has the correct set of components. The ECS scheduler automatically discovers and executes the system.

A developer's interaction is therefore declarative:
```java
// In entity creation logic...

// By adding these components, the entity is automatically
// processed by PlayerProcessMovementSystem each tick.
entityBuilder.with(new Player(...));
entityBuilder.with(new Velocity(...));
entityBuilder.with(new BoundingBox(...));
entityBuilder.with(new CollisionResultComponent());
entityBuilder.with(new PositionDataComponent());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerProcessMovementSystem()`. The system is managed by the engine's lifecycle and must be retrieved from the central System Registry if a reference is ever needed. Direct instantiation will result in a non-functional object that is not registered with the scheduler.

- **Manual Invocation:** Do not call the tick method directly. This bypasses the ECS scheduler's state management, concurrency control, and command buffer processing. Doing so will lead to unpredictable behavior, state corruption, and likely server crashes.

- **Component Omission:** Creating a player entity without all the components specified in the system's query (e.g., forgetting CollisionResultComponent) will cause the entity to be completely ignored by this system, resulting in a player who cannot move or collide with the world.

## Data Pipeline
The flow of data for a single entity during one execution of the tick method is linear and well-defined. The system transforms velocity and position data into updated state and deferred gameplay events.

> Flow:
> 1.  **Input:** Reads `TransformComponent`, `Velocity`, and `CollisionResultComponent` from the current entity's ArchetypeChunk.
> 2.  **Sanitization:** Performs a sanity check on the proposed movement distance to prevent physics exploits or glitches from large lag spikes.
> 3.  **Delegation:** Passes the entity's `BoundingBox`, current position, and intended movement vector to the global `CollisionModule.findIntersections`.
> 4.  **Processing:** The `CollisionModule` returns a populated `CollisionResult` object detailing any intersections.
> 5.  **State Update:** The system updates the entity's `PositionDataComponent` with information about the block it is standing on and inside. The player's velocity is processed based on the collision results.
> 6.  **Command Scheduling:** Any gameplay consequences, such as block trigger activation or environmental damage, are not executed immediately. Instead, they are wrapped in lambdas and submitted to the `CommandBuffer`. These commands are executed by the scheduler at the end of the tick to ensure world state remains consistent during system execution.

