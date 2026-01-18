---
description: Architectural reference for SimplePhysicsProvider
---

# SimplePhysicsProvider

**Package:** com.hypixel.hytale.server.core.modules.physics
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class SimplePhysicsProvider implements IBlockCollisionConsumer {
```

## Architecture & Concepts

The SimplePhysicsProvider is a stateful, single-entity physics simulation controller. It is not a full physics engine but rather a concrete implementation for handling the motion of simple, non-complex entities, such as projectiles. Its primary role is to integrate forces over a discrete time step (a "tick") and resolve collisions against the world and other entities.

This class operates as a callback target for the collision detection system, implementing the **IBlockCollisionConsumer** interface. It uses helper services like **BlockCollisionProvider** and **EntityCollisionProvider** to perform spatial queries (casts), but it contains the core logic for responding to the results of those queries.

Architecturally, it follows a classic game physics loop pattern for a single object:
1.  **Apply Forces:** Accumulate forces like gravity, drag, and external impulses.
2.  **Integrate:** Use a numerical integrator (**PhysicsBodyStateUpdaterSymplecticEuler**) to calculate the change in position and velocity over the time step.
3.  **Detect & Resolve:** Cast the entity's bounding box along its proposed movement vector to detect and resolve collisions.
4.  **Update State:** Write the final, resolved position and velocity back to the entity's components.

The provider maintains an internal state machine (**STATE**: Active, Resting, Inactive) to optimize performance by putting non-moving objects to "sleep" until an external force acts upon them.

**WARNING:** This class is marked as **@Deprecated**. It should not be used for new development and is subject to removal in future versions. It likely represents a legacy approach to physics simulation.

## Lifecycle & Ownership

-   **Creation:** An instance of SimplePhysicsProvider is typically created and owned by a component attached to a server-side Entity. The `initialize` method, which consumes a `Projectile` configuration object, indicates it is primarily designed for projectile entities at the moment of their spawning.
-   **Scope:** The lifecycle of a SimplePhysicsProvider instance is tightly coupled to the entity it simulates. It persists as long as the entity exists and requires physics updates. It is not shared between entities.
-   **Destruction:** The object is eligible for garbage collection when the owning entity is destroyed and its components are cleaned up. There is no explicit destruction or cleanup method.

## Internal State & Concurrency

-   **State:** This class is highly **mutable** and stateful. It maintains numerous fields representing the physics state for a single simulation frame, including `position`, `velocity`, `movement`, `onGround`, and `inFluid`. The `tick` method is a complex state transition function that modifies this internal state extensively before writing the final results back to the entity.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be operated exclusively by a single thread, typically the main server thread responsible for updating a specific world or region. Concurrent calls to the `tick` method will lead to race conditions, state corruption, and unpredictable physics behavior. All interactions with an instance must be synchronized externally or confined to a single-threaded context.

## API Surface

The public API is primarily for configuration and executing the simulation tick.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, entityVelocity, ...) | Entity | O(N) | Executes a single physics simulation step. Complexity depends on the number of potential colliders (N) in the entity's vicinity. Returns the entity it collided with, or null. |
| initialize(projectile, boundingBox) | void | O(1) | Configures the provider's physical properties (gravity, bounciness, drag) from a Projectile asset. This must be called before the first tick. |
| onCollision(...) | Result | O(1) | Internal callback method invoked by the BlockCollisionProvider. Contains the core logic for reacting to block collisions. |
| setGravity(gravity, boundingBox) | void | O(1) | Sets the gravitational acceleration and re-computes related drag factors. |
| setBounciness(bounciness) | void | O(1) | Sets the coefficient of restitution for collisions. |
| setResting(resting) | void | O(1) | Manually forces the provider into or out of the Resting state. |
| isResting() | boolean | O(1) | Returns true if the provider is in the Resting state, indicating it has ceased movement. |

## Integration Patterns

### Standard Usage

The provider is intended to be held by a physics-related component on an entity. The component's update logic calls `tick` once per server frame, passing in the required entity components for the provider to read from and write to.

```java
// Within an entity's physics component update method
SimplePhysicsProvider physicsProvider = this.getPhysicsProvider();
TransformComponent transform = entity.getComponent(TransformComponent.class);
Velocity velocity = entity.getComponent(Velocity.class);

// The tick method reads from and writes to the transform and velocity components
Entity collidedEntity = physicsProvider.tick(deltaTime, velocity, world, transform, selfRef, accessor);

if (collidedEntity != null) {
    // Handle entity-on-entity collision logic
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not share a single SimplePhysicsProvider instance across multiple entities. Its internal state is tied to one simulation and re-using it will cause catastrophic state corruption.
-   **Multi-threaded Access:** Never call `tick` or any `set` methods from multiple threads simultaneously. The object is not designed for concurrent access.
-   **Skipping Initialization:** Failure to call `initialize` will result in the provider operating with default, likely incorrect, physical properties (e.g., zero gravity, no bounciness).
-   **Direct State Manipulation:** Modifying the public `velocity` or `position` vectors from outside the class during a tick is not recommended. Use methods like `addVelocity` to apply forces for the *next* tick.

## Data Pipeline

The `tick` method orchestrates a precise flow of data to simulate one frame of motion.

> Flow:
> 1.  **Input:** Entity `TransformComponent`, `Velocity` component, `World` reference, and delta time (`dt`).
> 2.  **State Copy:** The entity's current position and velocity are copied into the provider's internal state vectors.
> 3.  **Force Integration:** External forces and impulses are applied. The `PhysicsBodyStateUpdater` calculates a predicted new position and velocity based on the current state and `dt`.
> 4.  **Collision Query:** The `BlockCollisionProvider` and `EntityCollisionProvider` are used to perform a shape-cast along the proposed movement vector.
> 5.  **Collision Response:** The `onCollision` callback is invoked for any detected hits. This logic may alter the final velocity (e.g., bouncing) and clamp the final position to the point of impact. Fluid physics are also calculated here.
> 6.  **Rotation Update:** The entity's rotation is updated to align with its new velocity vector, according to the configured `ROTATION_MODE`.
> 7.  **Output:** The final, resolved position and velocity are written back into the source `TransformComponent` and `Velocity` component.

