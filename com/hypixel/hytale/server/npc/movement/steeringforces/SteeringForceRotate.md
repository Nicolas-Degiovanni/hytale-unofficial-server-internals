---
description: Architectural reference for SteeringForceRotate
---

# SteeringForceRotate

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient

## Definition
```java
// Signature
public class SteeringForceRotate implements SteeringForce {
```

## Architecture & Concepts
The SteeringForceRotate class is a concrete implementation of the **Strategy** design pattern, where the SteeringForce interface defines the contract for a single, atomic unit of movement logic. This class encapsulates the specific behavior of rotating an entity to face a desired direction.

It serves as a low-level component within the server-side NPC AI and movement system. Its primary responsibility is not to execute the rotation itself, but to calculate the *intent* to rotate. It determines if an entity's current heading is outside an acceptable tolerance of a desired heading and, if so, populates an output Steering object with the target orientation.

This component acts as a bridge between high-level AI goals (e.g., "face the player") and the physics system. A controlling system, such as a SteeringBehavior, aggregates the output from this and other SteeringForce components to produce a final movement vector that is then applied to the entity's physics simulation.

## Lifecycle & Ownership
- **Creation:** SteeringForceRotate is a short-lived object. It is typically instantiated on-demand by a higher-level AI behavior or task for the duration of a single decision-making cycle. It is not managed by a central registry or dependency injection framework.
- **Scope:** The lifetime of a SteeringForceRotate instance is extremely brief, usually confined to a single method scope within one update tick of an AI controller.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the AI update method that created it completes, and the local reference goes out of scope. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** This class is highly **mutable** and stateful. Its internal fields for desiredHeading, heading, and tolerance are designed to be configured externally before each computation. The object acts as a temporary container for the parameters of a single, specific rotation calculation.

- **Thread Safety:** **This class is not thread-safe.** It is designed exclusively for use within a single thread, typically the server's main game loop or a dedicated AI thread for a specific world region. Its mutable state makes it fundamentally unsafe for concurrent access.

    **WARNING:** Do not share instances of SteeringForceRotate across threads. Do not cache and reuse instances across different AI agents without re-initializing all state fields.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compute(Steering output) | boolean | O(1) | Core logic. Calculates if rotation is needed. Populates the output parameter with the target yaw if the turn angle exceeds the tolerance. Returns true if a turn is required. |
| setDesiredHeading(float) | void | O(1) | Sets the target orientation in radians. |
| setHeading(float) | void | O(1) | Sets the current entity orientation in radians. |
| setHeading(Ref, Entity, ComponentAccessor) | void | O(1) | Convenience method to extract the current yaw from an entity's TransformComponent via the ECS. |
| setTolerance(double) | void | O(1) | Configures the acceptable angular deviation in radians before a turn is triggered. |

## Integration Patterns

### Standard Usage
SteeringForceRotate is intended to be used as a transient calculator within a more complex AI behavior. The standard pattern involves creating a new instance, configuring it with the entity's current state and the AI's goal, executing the computation, and then using the result.

```java
// Within an AI behavior's update method
Steering outputSteering = new Steering();
SteeringForceRotate rotationForce = new SteeringForceRotate();

// Configure the force with the AI's goal
float targetYaw = calculateYawToFaceTarget(target);
rotationForce.setDesiredHeading(targetYaw);

// Configure the force with the NPC's current state
// The ECS helper is the preferred method
rotationForce.setHeading(entityRef, npcEntity, componentAccessor);

// Execute the calculation
boolean needsToTurn = rotationForce.compute(outputSteering);

if (needsToTurn) {
    // The outputSteering object now contains the desired yaw.
    // This result is then passed to a movement aggregator.
    movementController.applySteering(outputSteering);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not store a SteeringForceRotate instance as a field in an AI behavior class and reuse it across multiple ticks without resetting its state. This can lead to using stale heading or tolerance data from a previous frame, causing erratic rotation.
- **Ignoring the Output Parameter:** The boolean return value of compute only indicates *if* a turn is needed. The actual target yaw is written to the Steering object passed as a parameter. Code that only checks the boolean and assumes a fixed rotation will not function correctly.
- **Direct State Manipulation:** While the fields are public for configuration, they should not be manipulated after the call to compute. The object's purpose is fulfilled once compute returns.

## Data Pipeline
The flow of data through this component is linear and unidirectional, transforming an AI goal into a concrete steering instruction.

> Flow:
> AI Goal (e.g., target position) -> Yaw Calculation -> **SteeringForceRotate** (configured with current & desired yaw) -> `compute()` -> Populated `Steering` Object -> Steering Aggregator -> Entity Physics Update

