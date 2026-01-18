---
description: Architectural reference for SteeringForceAvoidCollision
---

# SteeringForceAvoidCollision

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient

## Definition
```java
// Signature
public class SteeringForceAvoidCollision extends SteeringForceWithGroup {
```

## Architecture & Concepts
The SteeringForceAvoidCollision class is a specialized, stateful component within the server-side NPC steering behavior system. Its primary function is to calculate a corrective steering vector that prevents an NPC from colliding with other moving entities. It does not handle static world geometry.

This class implements a form of predictive collision avoidance. Rather than simply reacting to proximity, it uses a "swept sphere" intersection test to calculate the *time to impact* with other entities, assuming constant velocities. It then generates a force to either slow the NPC down or steer it away from the most imminent predicted collision.

Architecturally, it acts as a pluggable module in a larger composite behavior system. A high-level AI controller instantiates and configures this force alongside others (like SteeringForceSeek or SteeringForceWander). The outputs of all active forces are blended to produce the NPC's final movement for a given server tick. Its parent, SteeringForceWithGroup, provides the framework for iterating over a collection of nearby entities.

## Lifecycle & Ownership
-   **Creation:** An instance of SteeringForceAvoidCollision is typically created by a higher-level movement or AI system responsible for a single NPC's update tick. It is not a shared service and should be instantiated on a per-NPC, per-tick basis. Object pooling is a recommended optimization to reduce garbage collection pressure.
-   **Scope:** The object's state is valid only for the duration of a single movement calculation cycle. Its lifecycle is ephemeral: configure, populate, compute, and discard/reset.
-   **Destruction:** The object becomes eligible for garbage collection once the final steering vector has been computed and applied. The `reset` method facilitates its reuse in object pools by clearing collision-specific state, preparing it for the next calculation.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. It maintains internal state representing the controlling NPC (self) and the most critical collision threat found so far. Fields like `collisionTime`, `colliderPosition`, and `otherReference` are updated progressively as more potential colliders are processed via the `add` method. This state is critical for determining the most immediate danger.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within an entity's update loop. Accessing an instance from multiple threads will lead to race conditions and unpredictable, erroneous steering behavior as the internal state becomes corrupted.

## API Surface
The public contract is designed as a state machine: `setSelf` -> `reset` -> `add` (zero or more times) -> `compute`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setSelf(ref, position, velocity, radius, accessor) | void | O(1) | Configures the force with the state of the NPC being controlled. This is the mandatory first step. |
| reset() | void | O(1) | Resets the internal collision state. Must be called before processing a new group of entities. |
| add(ref, commandBuffer) | void | O(1) | Evaluates a single potential colliding entity. Performs physics calculations and updates internal state if this entity represents a more immediate threat than any previously checked. |
| compute(output) | boolean | O(1) | Calculates the final avoidance vector based on the most imminent collision found during the `add` phase. Modifies the provided Steering object. Returns true if a steering force was applied. |

## Integration Patterns

### Standard Usage
The class is intended to be used in a specific sequence within the NPC movement system. The system first gathers a list of nearby entities, then uses this class to find the most immediate collision threat and calculate a response.

```java
// Standard integration within an NPC update loop
SteeringForceAvoidCollision avoidanceForce = new SteeringForceAvoidCollision();
Steering outputSteering = new Steering();

// 1. Configure the force for the current NPC
avoidanceForce.setSelf(npcRef, npcPosition, npcVelocity, npcRadius, componentAccessor);

// 2. Reset state for the new calculation cycle
avoidanceForce.reset();

// 3. Iterate through nearby entities and add them as potential colliders
for (Ref<EntityStore> otherEntity : nearbyEntities) {
    if (otherEntity.equals(npcRef)) continue;
    avoidanceForce.add(otherEntity, commandBuffer);
}

// 4. Compute the final avoidance steering vector
boolean avoidanceApplied = avoidanceForce.compute(outputSteering);

if (avoidanceApplied) {
    // The outputSteering object now contains the avoidance vector.
    // This vector is then blended with other steering forces.
}
```

### Anti-Patterns (Do NOT do this)
-   **State Leakage:** Reusing an instance for a new NPC or a new tick without calling `reset`. This will cause collision data from the previous calculation to "leak" into the new one, resulting in NPCs reacting to phantom threats.
-   **Concurrent Usage:** Sharing a single instance across multiple NPC update tasks running in parallel. This is a critical error that will corrupt the internal state and cause undefined behavior.
-   **Incorrect Invocation Order:** Calling `compute` before all potential colliders have been processed with `add`, or calling `add` before `setSelf`. This will result in an incomplete or incorrect calculation.

## Data Pipeline
The component processes entity data to produce a steering adjustment. It acts as a filter, identifying the single most important collision threat from a group of entities.

> Flow:
> NPC Physics State (Position, Velocity) -> **setSelf()**
> &
> List of Nearby Entities -> **[Loop] add()** -> Internal State (closest collision time & target) -> **compute()** -> Output Steering Vector

