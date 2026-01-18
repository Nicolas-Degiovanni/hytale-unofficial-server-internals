---
description: Architectural reference for RotateObjectSystem
---

# RotateObjectSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** System Processor

## Definition
```java
// Signature
public class RotateObjectSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The RotateObjectSystem is a core processor within the server-side Entity Component System (ECS) framework. Its single, focused responsibility is to apply a continuous yaw (Y-axis) rotation to entities that are designed to spin.

This class embodies the "System" pattern in ECS architecture. It is a stateless logic container that operates on a specific subset of entities, defined by its component query. By design, it only processes entities that possess a **RotateObjectComponent**. This decouples the rotation *behavior* from the entity *data*, allowing any entity to become a rotating object simply by having the correct component attached.

The system is invoked by the main server game loop on every tick, ensuring smooth and consistent rotation that is synchronized with the server's simulation rate. It directly manipulates the entity's TransformComponent, which is the canonical source of truth for its position, scale, and orientation in the world.

## Lifecycle & Ownership
- **Creation:** Instantiated once by a higher-level System Registry or Module Loader during server bootstrap. Its dependencies, the ComponentType handles for TransformComponent and RotateObjectComponent, are injected via its constructor. This class is never instantiated directly by gameplay code.
- **Scope:** Session-scoped. It persists for the entire lifetime of the server process. As a stateless processor, a single instance is sufficient to handle all relevant entities.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only during a full server shutdown when the central System Registry is cleared.

## Internal State & Concurrency
- **State:** The RotateObjectSystem is fundamentally **stateless**. Its fields are immutable references to component type definitions, established at construction. It holds no per-entity data and does not cache any information between invocations of its tick method. All state is read from and written to the components passed into the tick method.
- **Thread Safety:** This system is not thread-safe for concurrent invocation. It is designed to be executed by a single, dedicated game loop thread per world instance. The ECS scheduler guarantees that systems are ticked in a deterministic, single-threaded sequence for a given set of components.

    **WARNING:** Manually invoking the tick method from an external thread will bypass engine safeguards and lead to severe race conditions and data corruption within the underlying component stores.

## API Surface
The public API is intended for consumption by the ECS engine, not by end-users.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Engine callback to declare interest in entities with a RotateObjectComponent. |
| tick(...) | void | O(1) per entity | Engine callback to execute rotation logic for a single entity in a matching chunk. |

## Integration Patterns

### Standard Usage
A developer never interacts with this system directly. To make an entity rotate, one must add the required components to it, typically during its creation. The engine will then automatically route the entity to this system for processing.

```java
// Example: Spawning a rotating entity
// This code would exist in a separate entity spawning service.

Entity entity = world.createEntity();

// Add a TransformComponent to give the entity a presence in the world.
entity.addComponent(new TransformComponent());

// Add a RotateObjectComponent to make it spin.
// The RotateObjectSystem will now process this entity every tick.
RotateObjectComponent rotator = new RotateObjectComponent();
rotator.setRotationSpeed(90.0f); // 90 degrees per second
entity.addComponent(rotator);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RotateObjectSystem()`. The server's System Registry is solely responsible for its lifecycle. Direct instantiation will result in a non-functional system that is not registered with the game loop.
- **Manual Invocation:** Never call the `tick` method directly. This bypasses the ECS scheduler, breaks frame-rate independence (delta time), and will cause unpredictable behavior or state corruption.
- **Missing TransformComponent:** Adding a RotateObjectComponent to an entity that lacks a TransformComponent is an invalid state. The system's internal assertions will fail, leading to a server crash. An entity must have a transform to be rotated.

## Data Pipeline
The system acts as a simple processor in the main game tick pipeline. It reads from two components and writes to one.

> Flow:
> Game Loop Tick -> ECS Scheduler -> **RotateObjectSystem.tick()** -> Reads `RotateObjectComponent` (for speed) & `TransformComponent` (for current rotation) -> Mutates `TransformComponent` (with new rotation) -> Downstream Network/Physics Systems

