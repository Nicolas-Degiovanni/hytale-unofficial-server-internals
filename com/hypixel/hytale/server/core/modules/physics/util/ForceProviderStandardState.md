---
description: Architectural reference for ForceProviderStandardState
---

# ForceProviderStandardState

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Transient

## Definition
```java
// Signature
public class ForceProviderStandardState {
```

## Architecture & Concepts
The ForceProviderStandardState class is a transient, mutable data structure that serves as a state accumulator for the physics engine. It is not a service or a manager, but rather a per-entity, per-tick container for all external physical influences.

Its primary role is to decouple various force-producing systems (e.g., gravity, character controllers, explosions, fluid dynamics) from the core physics integrator. Instead of directly modifying an entity's final velocity or force vectors, these systems contribute their influences—as forces, accelerations, impulses, or direct velocity changes—to an instance of this class.

At the end of the force accumulation phase of a physics tick, this state object is processed. The `convertToForces` method normalizes time-dependent inputs like acceleration and impulse into a single, unified force vector. The `updateVelocity` method applies direct, discrete changes to the entity's velocity. This design pattern centralizes the application of physics inputs, ensuring a predictable order of operations and simplifying the logic of individual force-producing systems.

### Lifecycle & Ownership
- **Creation:** Instantiated on the stack or as a short-lived heap object at the beginning of a physics simulation step for a single entity. It is typically created by the primary physics loop responsible for updating an entity's state.
- **Scope:** The object's lifetime is strictly confined to a single physics tick for one entity. It is intended to be discarded immediately after the entity's new velocity and position have been calculated.
- **Destruction:** As a transient object with no external references beyond the current tick, it is reclaimed by the garbage collector. There is no explicit cleanup or destruction logic.

## Internal State & Concurrency
- **State:** Highly **Mutable**. All public fields are designed for direct modification by external systems. This class acts as a temporary scratchpad for physics calculations, prioritizing performance over encapsulation.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. Its design with public, mutable fields makes it inherently vulnerable to race conditions. All operations on a given instance must be confined to the single thread responsible for processing the associated entity's physics for that tick.

## API Surface
The public contract includes both methods and directly accessible fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| displacedMass | double | O(1) | Public field representing the mass of fluid displaced by the entity, used for buoyancy calculations. |
| dragCoefficient | double | O(1) | Public field representing the entity's drag, used for air/fluid resistance calculations. |
| gravity | double | O(1) | Public field for the gravitational acceleration to be applied to the entity. |
| externalForce | Vector3d | O(1) | Public field accumulating forces to be applied over the tick. This is the primary output of the `convertToForces` method. |
| externalImpulse | Vector3d | O(1) | Public field for instantaneous changes in momentum. Converted to a force in `convertToForces`. |
| convertToForces(dt, mass) | void | O(1) | Normalizes `externalAcceleration` and `externalImpulse` into the `externalForce` vector based on delta-time and mass. Resets the source vectors to zero. |
| updateVelocity(velocity) | void | O(1) | Applies `nextTickVelocity` and `externalVelocity` to the entity's main velocity vector. Resets the source vectors. |
| clear() | void | O(1) | Resets the accumulated `externalForce` vector to zero. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a new instance for each entity per physics tick, populate its fields from various game systems, and then use it to compute the final forces and velocity adjustments.

```java
// Within a per-entity physics update loop
ForceProviderStandardState physicsState = new ForceProviderStandardState();

// 1. Game systems contribute to the state
applyGravity(entity, physicsState);
applyPlayerInput(entity, physicsState);
applyExplosionForces(entity, physicsState);

// 2. The state is processed to calculate final forces
physicsState.convertToForces(tickDeltaTime, entity.getMass());

// 3. The final force is used by the physics integrator
Vector3d totalForce = physicsState.externalForce;
// ... integrator applies totalForce to entity's acceleration

// 4. Direct velocity changes are applied
physicsState.updateVelocity(entity.getVelocity());
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not re-use a ForceProviderStandardState instance across multiple physics ticks without a full reset. Failure to do so will cause forces from a previous tick to be applied again, leading to incorrect simulation.
- **Instance Sharing:** Never share a single instance across multiple entities within the same tick. This will incorrectly merge forces intended for different objects.
- **Concurrent Modification:** Do not allow multiple threads to write to the same instance. This will lead to race conditions and non-deterministic physics behavior.

## Data Pipeline
This class acts as a collection point where disparate physics inputs are unified before being consumed by the core physics solver.

> Flow:
> Game Systems (Gravity, Input, etc.) -> Populate fields of **ForceProviderStandardState** -> Physics Integrator calls `convertToForces` -> Physics Integrator reads `externalForce` -> Entity State Update

