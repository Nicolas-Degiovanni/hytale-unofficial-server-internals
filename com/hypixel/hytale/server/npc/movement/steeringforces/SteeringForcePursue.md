---
description: Architectural reference for SteeringForcePursue
---

# SteeringForcePursue

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient

## Definition
```java
// Signature
public class SteeringForcePursue extends SteeringForceWithTarget {
```

## Architecture & Concepts
SteeringForcePursue is a concrete implementation of a steering force used within the server-side NPC artificial intelligence system. Its primary function is to generate a velocity vector that directs an NPC towards a moving target, while intelligently managing its arrival to prevent overshooting.

This class embodies the "Arrival" steering behavior. Unlike a simple "Seek" behavior which applies maximum force until the target is reached, Pursue calculates a force that diminishes as the NPC enters a defined `slowdownDistance` radius around the target. The force becomes zero once the NPC is within the `stopDistance`, causing it to come to a smooth halt.

It operates as a modular component within a larger steering framework. An NPC's final movement is typically the result of blending multiple forces. For example, a SteeringForcePursue might be combined with a SteeringForceAvoidObstacles to create emergent behavior where an NPC chases a player while navigating around trees. The output of this class is not a final velocity, but a *desired* velocity change, which is then processed by an accumulator and applied to the NPC's physics state.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level AI behavior, such as a `PursueEntityTask` or a similar state within an NPC's state machine. It is created when an NPC is commanded to begin pursuing a specific target.
-   **Scope:** The lifecycle of a SteeringForcePursue instance is tightly coupled to the duration of the pursuit task. It is a short-lived object that persists only as long as the NPC is actively engaged in that specific behavior.
-   **Destruction:** The object is eligible for garbage collection as soon as the parent AI task is completed or interrupted. There are no external references to manage.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It caches configuration parameters like `stopDistance` and `slowdownDistance`, along with pre-calculated squared values and deltas to optimize the per-tick `compute` method. The parent class, SteeringForceWithTarget, holds the mutable state for the target and self positions.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads or NPCs. It is designed to be exclusively owned and operated by a single NPC's update logic within a single thread, typically the main server game loop or a dedicated AI thread. Unsynchronized, concurrent calls to `setDistances` and `compute` will result in race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compute(Steering output) | boolean | O(1) | Core logic. Calculates the pursuit vector and populates the output parameter. Returns false if no force should be applied (e.g., target is null or NPC has arrived). |
| setDistances(double slowdown, double stop) | void | O(1) | Configures the primary arrival parameters and pre-calculates internal values for optimization. |
| setStopDistance(double stopDistance) | void | O(1) | Convenience method to update the stopping distance. |
| setSlowdownDistance(double slowdownDistance) | void | O(1) | Convenience method to update the slowdown distance. |
| setFalloff(double falloff) | void | O(1) | Adjusts the exponent for the deceleration curve, controlling how sharply the NPC slows down. |

## Integration Patterns

### Standard Usage
This component is intended to be used by a controlling AI task. On each update tick, the task updates the force with the current positions of the NPC and its target, then calls compute to get the desired steering output.

```java
// Within an NPC's AI behavior update method
SteeringForcePursue pursuitForce = this.getOrCreatePursuitForce();

// 1. Configure the force's parameters (often done at initialization)
pursuitForce.setDistances(20.0, 2.0); // Slow down at 20 units, stop at 2

// 2. Update dynamic state every tick
pursuitForce.setSelfPosition(npc.getPosition());
pursuitForce.setTargetPosition(target.getPosition());

// 3. Compute the desired steering output
Steering steeringOutput = new Steering();
if (pursuitForce.compute(steeringOutput)) {
    // 4. Add the resulting force to the NPC's steering accumulator
    npc.getSteeringAccumulator().add(steeringOutput);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Sharing:** Never share a single SteeringForcePursue instance between multiple NPCs. Each NPC requires its own instance to maintain its unique state relative to its target.
-   **Invalid Configuration:** Setting `stopDistance` to a value greater than or equal to `slowdownDistance` will break the arrival logic and may lead to mathematical exceptions or erratic movement.
-   **Ignoring Return Value:** The `compute` method returns `false` when the NPC has arrived at its destination. Code that unconditionally uses the `steeringOutput` object without checking this return value may override other important forces, such as idle behaviors.

## Data Pipeline
The component transforms positional data into a directional steering vector with a dynamically scaled magnitude.

> Flow:
> NPC Position (Vector3d) & Target Position (Vector3d) -> **SteeringForcePursue.compute()** -> Steering Object (with scaled velocity vector) -> Steering Accumulator -> NPC Physics Controller

