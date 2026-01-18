---
description: Architectural reference for PickupItemSystem
---

# PickupItemSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** System Component

## Definition
```java
// Signature
public class PickupItemSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts

The PickupItemSystem is a behavioral system within the server-side Entity Component System (ECS) framework. Its sole responsibility is to implement the "attraction" animation of an item entity moving towards a target entity, typically a player. It is a classic example of a data-oriented design; the system itself is stateless and contains only logic. All state, such as the item's position and animation progress, is stored in components attached to the entities it processes.

This system operates on a specific subset of entities defined by its internal Query. It will only process entities that possess both a PickupItemComponent and a TransformComponent. During each server tick, the ECS scheduler invokes this system's *tick* method for every matching entity, allowing it to update the entity's position to create a smooth interpolation towards its target.

The system leverages a CommandBuffer to enact world-state changes, such as entity removal. This is a critical architectural pattern that defers mutations to a synchronization point at the end of the tick, ensuring deterministic behavior and preventing race conditions between systems.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's core module loader during world initialization. Its dependencies, the ComponentType handles, are injected by the ECS framework. This is not an object that developers should ever create manually.
- **Scope:** The instance persists for the entire lifetime of the server world. It is re-used across every tick.
- **Destruction:** The instance is destroyed and garbage collected when the server world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The PickupItemSystem instance is effectively immutable after construction. Its fields are final references to component type definitions and a pre-built query. All mutable state it operates on is external, located within the components of the entities it processes (e.g., the lifetime remaining in PickupItemComponent, the position in TransformComponent).

- **Thread Safety:** This system is **not thread-safe** and must not be invoked from multiple threads simultaneously. However, it is designed to operate safely within the Hytale ECS scheduler. The scheduler guarantees that the *tick* method is executed for a given chunk of entities on a single worker thread. The use of a CommandBuffer for all cross-entity interactions and entity lifecycle changes is the primary mechanism that ensures safe concurrency at the framework level.

## API Surface

The primary API is the contract with the ECS scheduler, not with a typical developer. Developers interact with this system implicitly by manipulating components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, ...) | void | O(1) | Executes one step of the pickup animation for a single entity. Called by the ECS scheduler. |
| getQuery() | Query | O(1) | Returns the query used by the ECS scheduler to identify entities this system should process. |

## Integration Patterns

### Standard Usage

A developer does not call this system directly. To make an item entity fly towards a target, one must add a correctly configured PickupItemComponent to that entity. The system will automatically discover and process it on the next server tick.

```java
// Example: Spawning a dropped item that will fly to a player
// Assume 'world' is the server world instance and 'playerRef' is a valid Ref to a player entity.

// 1. Create the item entity
Ref<EntityStore> itemEntity = world.createEntity(itemArchetype);

// 2. Create the component that triggers the PickupItemSystem
PickupItemComponent pickup = new PickupItemComponent();
pickup.setTargetRef(playerRef);
pickup.setStartPosition(itemEntity.getPosition()); // Set the origin point for the animation
pickup.setLifeTime(0.5f); // Animation duration in seconds

// 3. Add the component to the entity. The system will now process it.
world.getCommandBuffer().addComponent(itemEntity, pickup);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PickupItemSystem()`. The ECS framework manages its lifecycle. Manual creation will result in a non-functional system that is not registered with the scheduler.
- **Manual Invocation:** Do not call the *tick* method directly. This bypasses the ECS scheduler, the CommandBuffer synchronization, and thread-safety mechanisms, which will lead to world corruption, race conditions, and crashes.
- **External State Modification:** Modifying an entity's TransformComponent from another thread while it is being processed by this system can lead to visual stuttering or race conditions. All interactions should be done via components and systems.

## Data Pipeline

The system's operation during a single tick follows a clear data flow, transforming component data into new component data and lifecycle commands.

> Flow:
> ECS Scheduler identifies an entity with PickupItemComponent & TransformComponent -> **PickupItemSystem.tick()** is invoked -> Reads item's TransformComponent -> Reads target's TransformComponent -> Calculates new interpolated position -> Writes new position to item's TransformComponent -> If animation is finished, writes `removeEntity` to the CommandBuffer -> ECS Sync Point executes commands.

