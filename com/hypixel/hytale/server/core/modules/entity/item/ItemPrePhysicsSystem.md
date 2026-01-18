---
description: Architectural reference for ItemPrePhysicsSystem
---

# ItemPrePhysicsSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** System Component

## Definition
```java
// Signature
public class ItemPrePhysicsSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The ItemPrePhysicsSystem is a specialized system within the server's Entity Component System (ECS) framework. It executes during the initial phase of the physics simulation stage for each game tick. Its primary architectural role is to prepare item entities for the main collision and movement physics pipeline by performing corrective and foundational physics calculations.

This system isolates two critical concerns:
1.  **State Correction:** It ensures item entities are not embedded inside solid blocks, a common issue when items are spawned or land due to high velocity. This "un-sticking" logic is a crucial prerequisite for stable collision detection.
2.  **Gravity Simulation:** It applies the fundamental force of gravity to item entities, updating their vertical velocity. This calculation is separated to ensure it is applied consistently before any other forces or collisions are resolved in the same tick.

The system operates exclusively on entities that possess the following components: ItemComponent, TransformComponent, BoundingBox, Velocity, and PhysicsValues. This explicit contract is defined by its internal ECS query, ensuring it only processes relevant entities and avoids side effects on other entity types.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's central ECS scheduler during server bootstrap. Its dependencies, which are references to various ComponentTypes, are injected by the framework at this time.
-   **Scope:** Session-scoped. The single instance of this system persists for the entire lifetime of the server process.
-   **Destruction:** The system is decommissioned and becomes eligible for garbage collection only when the server shuts down. There is no manual destruction logic.

## Internal State & Concurrency
-   **State:** The ItemPrePhysicsSystem is fundamentally **stateless**. Its member fields are final references to component definitions and the entity query, which are established at construction and never change. All state modifications performed by this system are applied directly to the components of the entities it processes.

-   **Thread Safety:** This system is designed for parallel execution and is considered **thread-safe**. The `isParallel` method allows the ECS scheduler to distribute the workload across multiple threads. Safety is guaranteed because:
    -   The system itself holds no mutable state.
    -   The ECS scheduler ensures that each worker thread processes a distinct set of entities (or ArchetypeChunks), preventing two threads from modifying the same entity's components simultaneously.
    -   The static utility methods, `moveOutOfBlock` and `applyGravity`, are pure functions that operate exclusively on their arguments without external side effects.

## API Surface
The primary API is the `tick` method, which is invoked by the ECS scheduler, not by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, archetypeChunk, store, commandBuffer) | void | O(1) | Executes one iteration of the system's logic for a single entity. Modifies the entity's Velocity and potentially its TransformComponent. This is the core entry point for the ECS scheduler. |
| getQuery() | Query | O(1) | Returns the immutable ECS query that defines which entities this system will process. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. It is an automatic part of the server's physics pipeline. To have an entity be processed by this system, a developer must construct an entity with the required set of components.

```java
// Example of creating an entity that this system will process
Entity entity = entityStore.createEntity();

// The presence of these components makes the entity eligible
commandBuffer.addComponent(entity, new ItemComponent(...));
commandBuffer.addComponent(entity, new TransformComponent(...));
commandBuffer.addComponent(entity, new BoundingBox(...));
commandBuffer.addComponent(entity, new Velocity(...));
commandBuffer.addComponent(entity, new PhysicsValues(...));

// The ItemPrePhysicsSystem will automatically find and tick this entity.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ItemPrePhysicsSystem()`. The ECS framework is responsible for its lifecycle and dependency injection. Manual creation will result in a non-functional system that is not registered with the scheduler.
-   **Manual Invocation:** Do not call the `tick` method directly. This bypasses the scheduler, breaks parallelization guarantees, and can cause severe race conditions or out-of-order physics calculations if other systems are not executed in the correct sequence.
-   **Component Omission:** Creating an item entity without all the required components (e.g., forgetting PhysicsValues) will cause the system's query to ignore it, resulting in an item that does not experience gravity or un-sticking behavior.

## Data Pipeline
The ItemPrePhysicsSystem acts as an early-stage processor in the server's per-tick physics pipeline. It reads world state and entity state, modifies entity components, and passes the updated state to subsequent systems.

> **Flow:**
> ECS Scheduler identifies entities with required components -> **ItemPrePhysicsSystem.tick()** reads Transform, BoundingBox, and WorldChunk data -> Logic calculates necessary adjustments -> System mutates the entity's **Velocity** and **TransformComponent** -> Data flows to downstream systems like CollisionDetectionSystem and MovementSystem.

