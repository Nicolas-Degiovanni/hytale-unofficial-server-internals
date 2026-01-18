---
description: Architectural reference for BodyMotionFlock
---

# BodyMotionFlock

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionFlock extends BodyMotionBase {
```

## Architecture & Concepts
The BodyMotionFlock class is a specialized motion controller responsible for implementing flocking behavior for server-side NPCs. It operates as a concrete strategy within the broader NPC AI and movement system, selected by an NPC's Role when it needs to exhibit group movement dynamics.

This class translates the state of a group of entities into a low-level steering command for a single NPC. It implements a variation of the classic "boids" algorithm by calculating three primary steering forces:
1.  **Cohesion:** Steering towards the average position of the flockmates.
2.  **Separation:** Steering away from flockmates that are too close.
3.  **Alignment/Leader Following:** Steering in the general direction of the flock's movement and also towards the designated leader of the group.

It is not a standalone system; it is deeply integrated with the Entity Component System (ECS). It queries for components like FlockMembership, EntityGroup, and TransformComponent to gather the necessary spatial and relational data to perform its calculations. The core logic is orchestrated within the computeSteering method, which uses a GroupSteeringAccumulator helper object to efficiently iterate over flock members and aggregate their states.

## Lifecycle & Ownership
-   **Creation:** An instance of BodyMotionFlock is created via its corresponding builder, BuilderBodyMotionFlock. This typically occurs during the configuration and instantiation of an NPC's AI profile or Role. It is not intended to be created dynamically during the game loop.
-   **Scope:** The object's lifetime is bound to the NPC entity that owns it. It persists as long as the NPC's current Role requires flocking behavior. If the NPC changes roles to one that does not use flocking, this object may be replaced or discarded.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC or its associated Role is destroyed or reconfigured. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
-   **State:** BodyMotionFlock is **stateful and mutable**. Its primary internal state is the GroupSteeringAccumulator instance. This accumulator is reset and repopulated with data from the flock during every call to computeSteering, making the method non-idempotent across different game ticks.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread responsible for ticking the world and its entities. The mutable state of the GroupSteeringAccumulator makes concurrent calls to computeSteering on the same instance unsafe, which would lead to race conditions and corrupted steering calculations.

## API Surface
The public contract is focused entirely on the steering computation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(ref, role, sensorInfo, dt, desiredSteering, componentAccessor) | boolean | O(N) | Calculates the flocking steering vector. N is the number of entities in the flock. Populates the output parameter desiredSteering. Returns false if the entity is not in a valid flock, otherwise true. |

## Integration Patterns

### Standard Usage
This class is intended to be invoked by a higher-level AI system, such as an NPC's Role, once per simulation tick. The system retrieves the active motion controller and passes it the required context to calculate the next movement step.

```java
// Within an NPC's Role update method
BodyMotionFlock motionController = this.getActiveMotionController();
Steering outputSteering = new Steering();

// The method populates the outputSteering object
boolean success = motionController.computeSteering(
    npcRef,
    this,
    sensorInfo,
    deltaTime,
    outputSteering,
    componentAccessor
);

if (success) {
    // Pass outputSteering to the physics/movement system
    npcMovementComponent.applySteering(outputSteering);
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Reuse:** Do not hold onto the calculated Steering object or the internal state of the GroupSteeringAccumulator across ticks. The system is designed to be re-evaluated from scratch each frame based on the current world state.
-   **Concurrent Execution:** Never call computeSteering from multiple threads on the same BodyMotionFlock instance. All NPC AI and motion logic for a given world region should be confined to a single update thread.
-   **Manual Instantiation:** Avoid creating this object directly. It should be constructed and managed by the NPC's configuration and Role systems to ensure its parameters (like influence range) are set correctly.

## Data Pipeline
The flow of data through this component is a classic example of an ECS pattern: gather data from components, process it, and output a result that influences other systems.

> Flow:
> NPC Update Tick -> Role requests steering -> **BodyMotionFlock.computeSteering** -> Accesses FlockMembership & EntityGroup components -> Iterates members via GroupSteeringAccumulator -> Fetches TransformComponent for each member -> Computes aggregate vectors -> Populates `Steering` object -> `Steering` object consumed by Movement System

