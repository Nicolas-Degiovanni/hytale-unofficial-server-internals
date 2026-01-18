---
description: Architectural reference for MotionControllerBase
---

# MotionControllerBase

**Package:** com.hypixel.hytale.server.npc.movement.controllers
**Type:** Transient State Object

## Definition
```java
// Signature
public abstract class MotionControllerBase implements MotionController {
```

## Architecture & Concepts
MotionControllerBase is an abstract foundational class within the server-side NPC movement system. It serves as the primary engine for translating high-level behavioral intentions into low-level physical simulation. This class acts as the bridge between the AI layer, represented by a **Role** and its **Steering** outputs, and the world simulation layer, which involves collision detection, physics, and entity transformation.

Architecturally, it embodies a stateful strategy pattern. Each concrete implementation (e.g., for walking, flying, or swimming) provides a specific strategy for movement, while this base class provides the common infrastructure for physics integration, state management, and entity updates.

Its core responsibility is to process a single movement tick. It takes a desired translation and rotation from the **Steering** object, applies physical forces like gravity and knockback, resolves collisions with the world geometry, and computes the final valid position and orientation for the NPC. It directly manipulates the entity's **TransformComponent** and **HeadRotation** component, making it a critical link in the entity-component-system (ECS) architecture of the server.

**Key Concepts:**
- **Steering Decoupling:** It decouples the AI's *intent* to move (Steering) from the *execution* of that movement, allowing for complex physics without burdening the behavior tree or state machine logic.
- **Force Accumulation:** It manages both AI-driven movement and external physical impulses (e.g., from damage or environmental effects) through a unified force and velocity model (**forceVelocity**, **appliedVelocities**).
- **Collision Resolution:** It is the primary client of the **CollisionModule**, using it to validate potential moves and slide along surfaces. The **bisect** method is a key internal algorithm for finding the last valid position before a collision.
- **State Machine:** It internally manages a **MotionKind** (e.g., MOVING, SWIMMING, FLYING), which influences physics calculations and informs the animation system via the **MovementStates** component.

## Lifecycle & Ownership
- **Creation:** A MotionControllerBase instance is not created directly. Concrete subclasses are instantiated by a builder (**BuilderMotionControllerBase**) as part of an NPC's initialization pipeline, typically defined in asset files. The controller is tightly coupled to a single **NPCEntity**.
- **Scope:** The controller's lifetime is identical to that of the NPC it controls. It persists as a core component of the NPC's runtime state.
- **Destruction:** The object is eligible for garbage collection when the parent **NPCEntity** is despawned or destroyed. The **deactivate** method is called to signal the end of its active use, though it performs no explicit resource cleanup.

## Internal State & Concurrency
- **State:** This class is highly mutable and stateful. It maintains a comprehensive snapshot of the NPC's physical state, including:
    - **position**, **yaw**, **pitch**, **roll**: The current calculated position and orientation.
    - **forceVelocity**, **appliedVelocities**: Vectors representing external forces like knockback.
    - **collisionBoundingBox**: The NPC's physical hitbox for collision checks.
    - **collisionResult**: A cached object to store detailed results from the **CollisionModule**.
    - Numerous boolean flags (**isObstructed**, **requiresPreciseMovement**) that modify behavior on a per-tick basis.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be exclusively accessed and manipulated by the main server tick thread. All state modifications occur within the scope of a single tick update, primarily inside the **steer** method.

    **WARNING:** Concurrent access from other threads will lead to race conditions, physics glitches, and server instability. All interactions with a motion controller must be synchronized with the main game loop.

## API Surface
The public API is designed for interaction from the NPC's **Role** and core server systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steer(...) | double | O(N) | The main entry point for a single physics tick. Orchestrates all movement, collision, and state updates. Complexity is dependent on collision checks. |
| addForce(force, config) | void | O(1) | Applies an external impulse to the NPC, such as knockback from an attack. |
| forceVelocity(velocity, config, ignoreDamping) | void | O(1) | Immediately overrides the NPC's velocity. Used for effects like explosions or scripted movements. |
| updateMovementState(...) | void | O(1) | Updates the **MovementStates** component based on the current **MotionKind** and velocity. Critical for animations. |
| probeMove(...) | double | O(N) | A predictive method to test a potential move without actually executing it. Used by pathfinding systems. |
| isValidPosition(position, accessor) | boolean | O(N) | Checks if a given position in the world is free of collisions for this NPC. |
| activate() / deactivate() | void | O(1) | Lifecycle methods called when the controller becomes active or inactive. Resets internal state. |

## Integration Patterns

### Standard Usage
The controller is driven by a **Role** during the NPC's update cycle. The **Role** determines the desired movement and passes it to the controller via a **Steering** object.

```java
// Within an NPC's Role update method
Steering bodySteering = calculateBodySteering(context);
Steering headSteering = calculateHeadSteering(context);

// The steer method executes the full physics tick for this NPC
motionController.steer(
    entityRef,
    this, // the Role
    bodySteering,
    headSteering,
    tickInterval,
    componentAccessor
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ConcreteMotionController()`. Controllers must be constructed via their corresponding builders during NPC asset loading to ensure proper initialization.
- **External State Mutation:** Do not directly modify public fields like **position** or **forceVelocity** from outside the class. Use API methods like **addForce** or allow the **steer** method to manage state. Direct mutation will break the physics simulation.
- **Ignoring the Tick Cycle:** Calling methods like **steer** multiple times per tick or out of sequence with the main game loop will cause unpredictable behavior and desynchronization.

## Data Pipeline
The **steer** method orchestrates a complex data flow to process one frame of movement.

> Flow:
> **Role** (AI Layer) -> **Steering** object -> **steer()** method
> 1.  **readEntityPosition**: Synchronizes internal state with the entity's current **TransformComponent**.
> 2.  **computeMove** (abstract): The concrete implementation calculates a desired translation vector based on steering, gravity, and other forces.
> 3.  **executeMove** (abstract): The translation is passed to the **CollisionModule**, which resolves collisions and returns a final, valid translation vector.
> 4.  **processTriggers**: Checks for and activates any trigger volumes the NPC entered during its movement. May apply damage or other effects.
> 5.  **calculateYaw/Pitch/Roll**: Computes the final body and head orientation based on the movement vector and steering targets.
> 6.  **moveEntity**: Commits the final calculated position and rotation back to the entity's **TransformComponent**.
> 7.  **dampForceVelocity**: Applies drag and friction to external forces to simulate decay over time.

