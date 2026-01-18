---
description: Architectural reference for ItemPhysicsSystem
---

# ItemPhysicsSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** System Component

## Definition
```java
// Signature
public class ItemPhysicsSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts

The ItemPhysicsSystem is a core server-side system within the Entity Component System (ECS) framework responsible for simulating the physical behavior of item entities in the world. It operates exclusively on entities possessing a specific combination of components: ItemPhysicsComponent, BoundingBox, and Velocity.

Its primary function is to apply simplified physics, including movement derived from velocity and collision detection against the static world geometry (blocks). This system ensures that dropped items fall, slide, and come to rest on surfaces in a predictable, server-authoritative manner.

This system acts as a high-level orchestrator, delegating the complex and computationally expensive task of collision detection to the specialized CollisionModule. It consumes the results of these collision checks to update an entity's TransformComponent and Velocity, effectively moving the entity through the world.

For state changes that cannot be safely performed during an iteration, such as entity removal, the system utilizes a CommandBuffer. This is a standard ECS pattern that queues operations to be executed by the framework at the end of the tick, preventing iterator invalidation and race conditions.

## Lifecycle & Ownership

-   **Creation:** Instantiated by the server's central ECS scheduler during the world initialization phase. Its dependencies, such as required ComponentType handles, are resolved and injected by the framework. Direct instantiation by developers is not supported.
-   **Scope:** A single instance of this system persists for the entire lifetime of a server world. The system itself is stateless; its lifetime is tied to the server process, not to any specific entity.
-   **Destruction:** The system is decommissioned and marked for garbage collection when the server world is unloaded or the server process terminates.

## Internal State & Concurrency

-   **State:** The ItemPhysicsSystem is fundamentally **stateless**. It does not retain any data between ticks or across different entities. All state it operates on is read from and written to the components of the entities it processes. Temporary, mutable objects like CollisionResult are allocated and reset within the scope of a single `tick` method invocation.
-   **Thread Safety:** This system is **not thread-safe** and explicitly signals its requirement for serial execution. The `isParallel` method returns `false`, instructing the ECS scheduler to process all entities matching its query on a single thread. This design prevents data races when modifying shared component data and interacting with non-thread-safe services like the World.

**WARNING:** Any attempt to configure the ECS scheduler to run this system in parallel will result in severe state corruption, unpredictable physics, and server instability.

## API Surface

The public contract is defined by its implementation of the EntityTickingSystem interface. Direct calls to these methods are managed by the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the component query used by the scheduler to identify processable entities. |
| isParallel(...) | boolean | O(1) | Informs the scheduler that this system cannot be run in parallel. Always returns false. |
| tick(...) | void | O(C) | Executes one physics step for a single entity. Complexity is dominated by the CollisionModule. |

## Integration Patterns

### Standard Usage

Developers do not interact with this system directly. To subject an entity to item physics, a developer attaches the required components to it. The ECS framework automatically ensures the ItemPhysicsSystem will process the entity on each server tick.

```java
// Example: Spawning a new item entity
Entity entity = world.createEntity();

// The system will now automatically process this entity
entity.addComponent(new TransformComponent(spawnPosition));
entity.addComponent(new Velocity(initialVelocity));
entity.addComponent(new BoundingBox(itemBounds));
entity.addComponent(new ItemPhysicsComponent());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new ItemPhysicsSystem()`. The system's lifecycle is strictly managed by the ECS framework, which handles dependency injection.
-   **Manual Invocation:** Do not call the `tick` method directly. This bypasses the ECS scheduler, breaks the execution order, and will lead to unpredictable behavior and state corruption.
-   **Component Omission:** Creating an entity with an ItemPhysicsComponent but without Velocity or BoundingBox will cause the entity to be ignored by this system, resulting in it remaining static and unresponsive.

## Data Pipeline

The system processes data for a single entity within the scope of one server tick. The flow transforms the entity's positional and velocity data based on world collision.

> Flow:
> Entity State (TransformComponent, Velocity) -> **ItemPhysicsSystem.tick()** -> CollisionModule Query -> CollisionResult -> State Mutation (Updated TransformComponent, Updated Velocity) OR CommandBuffer.removeEntity()

