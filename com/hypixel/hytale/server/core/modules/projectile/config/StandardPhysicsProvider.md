---
description: Architectural reference for StandardPhysicsProvider
---

# StandardPhysicsProvider

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Component

## Definition
```java
// Signature
public class StandardPhysicsProvider implements IBlockCollisionConsumer, Component<EntityStore> {
```

## Architecture & Concepts

The StandardPhysicsProvider is a stateful component that encapsulates the entire physics simulation for a single entity, typically a projectile. It is a core building block of the server-side physics engine, designed to operate within an Entity-Component-System (ECS) architecture. Each instance is self-contained, managing an entity's position, velocity, and interactions with the game world.

This class acts as the primary integration point between abstract physics calculations and concrete game logic. It is responsible for:
- **Kinematics:** Applying forces such as gravity, drag, and buoyancy using a Symplectic Euler integration method.
- **Collision Detection:** Utilizing internal providers (BlockCollisionProvider, EntityRefCollisionProvider) to detect collisions with world geometry and other entities.
- **Collision Response:** Implementing logic for bouncing, sliding, and stopping upon contact with solid surfaces. It also handles fluid dynamics, calculating displaced mass and sub-surface volume when entering liquids.
- **Game Logic Triggering:** Firing high-level game events through the InteractionManager upon significant physics events, such as a projectile hitting a target or bouncing off a wall. This decouples the raw physics simulation from the consequences of those events.

It implements the IBlockCollisionConsumer interface, positioning itself as a direct callback target for the broader collision detection systems.

### Lifecycle & Ownership
- **Creation:** An instance is created and attached to an entity when that entity is spawned. This is typically managed by a higher-level system like the ProjectileModule. The constructor requires a complete physical definition, including a BoundingBox, a configuration object (StandardPhysicsConfig), and an initial force vector.
- **Scope:** The component's lifecycle is strictly tied to the entity it is attached to. It persists for the entire duration of the entity's existence, accumulating state changes (e.g., number of bounces) across multiple game ticks.
- **Destruction:** The component is destroyed when its parent entity is removed from the world. The internal ImpactConsumer and BounceConsumer logic may themselves trigger the entity's removal via a command buffer.

## Internal State & Concurrency
- **State:** This component is highly mutable. It maintains the complete physical state of an entity for the current tick, including position, velocity, movement vectors, fluid interaction state, and collision contact points. It also caches references to world services and collision providers.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively owned and manipulated by the main server thread during the physics update phase of the game loop. All fields are unprotected, and concurrent access will lead to state corruption, race conditions, and unpredictable physics behavior.

**WARNING:** Do not access or modify an instance of StandardPhysicsProvider from any thread other than the one executing the main world tick.

## API Surface

The public API is designed for a controlling system to drive the simulation tick-by-tick.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onCollision(...) | IBlockCollisionConsumer.Result | O(1) | Callback invoked by the collision system. Implements bounce, slide, and fluid entry logic. **WARNING:** This should only be called by the engine's collision pipeline. |
| finishTick(transform, velocity) | void | O(1) | Finalizes the physics tick, writing the computed position and velocity back to the entity's core components. Resets transient tick data. |
| rotateBody(dt, bodyRotation) | void | O(1) | Calculates and applies rotational changes to the entity's transform based on its velocity vector, according to the configured rotation mode. |
| isOnGround() | boolean | O(1) | Returns true if the physics body is currently considered to be resting on a solid surface. |
| isSwimming() | boolean | O(1) | Returns true if the physics body is considered to be swimming (i.e., fully submerged and stabilized in a fluid). |

## Integration Patterns

### Standard Usage

The component is intended to be managed by a dedicated physics system. The system retrieves the component from an entity, prepares it for the tick, executes the simulation steps, and writes the results back.

```java
// Within a server-side physics system update loop
TransformComponent transform = entity.getComponent(TransformComponent.class);
Velocity velocity = entity.getComponent(Velocity.class);
StandardPhysicsProvider physics = entity.getComponent(StandardPhysicsProvider.class);

// 1. Prepare for tick (not shown: setting world, position, etc.)
// 2. Execute physics simulation steps (e.g., collision checks, force integration)
// ... simulation logic updates internal state of 'physics' ...

// 3. Finalize the tick and update the entity's canonical state
physics.finishTick(transform, velocity);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new StandardPhysicsProvider()`. The component must be created and managed by the ECS framework to ensure it is correctly registered and initialized.
- **State Tampering:** Do not manually modify internal state vectors like `position` or `velocity` mid-tick. All state changes should be the result of the integrated force calculations and collision responses managed by the component's internal logic.
- **Asynchronous Access:** Do not retain a reference to this component and access it from another thread or a delayed task. Its state is only valid during the synchronous execution of a single game tick.

## Data Pipeline

The StandardPhysicsProvider processes data in a well-defined sequence within a single game tick.

> Flow:
> Entity State (Transform, Velocity) -> **StandardPhysicsProvider** (Force Integration) -> Collision System -> **StandardPhysicsProvider.onCollision** (Response) -> **StandardPhysicsProvider.finishTick** -> Entity State Update -> InteractionManager (Event Trigger)

