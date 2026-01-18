---
description: Architectural reference for MotionControllerDive
---

# MotionControllerDive

**Package:** com.hypixel.hytale.server.npc.movement.controllers
**Type:** Transient

## Definition
```java
// Signature
public class MotionControllerDive extends MotionControllerBase {
```

## Architecture & Concepts
The MotionControllerDive is a specialized, stateful component responsible for translating high-level AI steering commands into low-level physical movements for Non-Player Characters (NPCs) in aquatic environments. It is a concrete implementation of the strategy pattern, where different MotionController classes dictate how an NPC navigates the world.

This controller's primary domain is swimming and diving. It simulates physics such as acceleration, turning speed, gravity, and buoyancy. Its core responsibility is to calculate a valid translation vector each server tick based on input from a Steering object and then execute that movement, resolving any collisions with the world geometry.

It operates as a bridge between the abstract decision-making layer (AI Roles and Behaviors) and the concrete physics and collision layer (CollisionModule). It makes extensive use of the PositionProbeWater utility to continuously sample the environment, determining if the NPC is submerged, on the ground, or in the air. This environmental awareness is critical for its state transitions, for example, switching from ground movement to swimming.

Unlike simpler controllers, MotionControllerDive manages complex 3D movement, including pitch and yaw, to simulate realistic diving and surfacing behaviors.

### Lifecycle & Ownership
- **Creation:** MotionControllerDive is instantiated via its corresponding builder, BuilderMotionControllerDive. This process is typically managed by an NPC factory or configuration loader when an NPC with aquatic capabilities is spawned. It is not intended for direct instantiation.
- **Scope:** The lifecycle of a MotionControllerDive instance is tied to the NPC it controls. It persists as long as the NPC is active and its current behavior requires diving or swimming locomotion. It is replaced if the NPC's state machine transitions to a behavior requiring a different controller (e.g., MotionControllerWalk).
- **Destruction:** The object is eligible for garbage collection when the parent NPC is despawned or when it is replaced by another motion controller. There are no explicit cleanup methods; its state is self-contained and does not hold external resources that require manual release.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains numerous fields representing the NPC's current physical state, including `moveSpeed`, `climbSpeed`, `collisionWithSolid`, and environmental data cached by its `moveProbe`. This internal state is updated on every call to `computeMove` and `executeMove`.

- **Thread Safety:** **This class is not thread-safe and must be considered thread-hostile.** All interactions with a MotionControllerDive instance must be confined to the main server thread that executes the game tick. Its methods modify internal state without any synchronization mechanisms. Concurrent access from multiple threads will lead to race conditions, corrupted state, and unpredictable physics calculations.

## API Surface
The public API is designed for interaction with the server's NPC update loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeMove(...) | double | O(1) | Calculates the desired translation vector for the current tick based on steering input. This is the primary logic entry point. |
| executeMove(...) | double | O(C) | Applies the computed translation, detects collisions via the CollisionModule, and updates the NPC's final position. C represents the complexity of world collision queries. |
| getDesiredVerticalRange(...) | VerticalRange | O(1) | Provides the valid vertical movement bounds to the AI system, preventing it from generating paths that are too deep or too shallow. |
| canAct(...) | boolean | O(1) | A critical state check. Returns true only if the NPC is in a state where it can perform controlled movement (e.g., alive and in water). |
| probeMove(...) | double | O(C) | A predictive method used by the pathfinding system to test a potential move without actually executing it. Simulates movement and collision along a vector. |

## Integration Patterns

### Standard Usage
The controller is managed by the NPC's core logic. An AI behavior generates a Steering command, which is then fed into the controller during the NPC's update phase. The resulting translation is then applied.

```java
// Within an NPC's update loop
Steering steering = npc.getRole().getSteering();
Vector3d translation = new Vector3d();

// 1. Calculate the desired movement based on AI input
motionController.computeMove(ref, role, steering, dt, translation, accessor);

// 2. Apply the movement and resolve collisions
motionController.executeMove(ref, role, dt, translation, accessor);

// 3. Update the NPC's transform with the final, validated position
transform.setPosition(motionController.getPosition());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new MotionControllerDive()`. The object is complex and must be configured through its corresponding builder (BuilderMotionControllerDive) to ensure all physics parameters are correctly initialized.
- **State Tampering:** Do not modify public fields or internal state vectors (e.g., `position`, `moveSpeed`) directly from outside the class. All state changes must occur through the `computeMove` and `executeMove` cycle.
- **Multi-threaded Access:** Never call any method on this object from an asynchronous task or a different thread. All interactions must be serialized on the main server tick.

## Data Pipeline
The MotionControllerDive functions as a key processing stage in the NPC movement pipeline. It transforms abstract intent into a concrete, collision-aware position update.

> Flow:
> AI Behavior (e.g., PathFollower) -> **Steering** object (target direction, speed) -> **MotionControllerDive.computeMove** (calculates ideal translation) -> **MotionControllerDive.executeMove** (uses CollisionModule to find actual position) -> Updated **TransformComponent** (NPC moves in the world)

