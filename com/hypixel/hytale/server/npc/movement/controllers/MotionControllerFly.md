---
description: Architectural reference for MotionControllerFly
---

# MotionControllerFly

**Package:** com.hypixel.hytale.server.npc.movement.controllers
**Type:** Stateful Component

## Definition
```java
// Signature
public class MotionControllerFly extends MotionControllerBase {
```

## Architecture & Concepts
The MotionControllerFly is a specialized, server-side component responsible for calculating the three-dimensional movement physics for flying Non-Player Characters (NPCs). It serves as a concrete implementation within the broader NPC movement framework, translating high-level behavioral intent into low-level, physically plausible motion.

This class acts as the bridge between an NPC's AI decision-making layer (which populates a Steering object) and the world's physics and collision systems. It consumes a desired travel vector and speed from the Steering object and computes a final translation vector for the current tick. This computation is complex, accounting for aerodynamic constraints such as acceleration, turning speed, roll, pitch limitations, and gravity.

It is a core component for any entity that requires flight capabilities. It directly interacts with the CollisionModule to ensure that all calculated movements are validated against world geometry, preventing NPCs from flying through solid objects. Its internal state machine manages transitions between flying, falling, and being idle, based on environmental probes and AI commands.

### Lifecycle & Ownership
- **Creation:** MotionControllerFly is not instantiated directly. It is constructed via its corresponding builder, BuilderMotionControllerFly, as part of the NPC asset loading and instantiation process. This ensures that all physical parameters (e.g., max speed, turn radius) are correctly configured from game data.
- **Scope:** The lifecycle of a MotionControllerFly instance is tightly coupled to the NPC entity it governs. It is created when the NPC is spawned into the world and persists for the entire duration of the NPC's existence.
- **Destruction:** The object is eligible for garbage collection when its owning NPC entity is despawned or destroyed. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains critical state across ticks, including lastVelocity, lastSpeed, and lastRoll. This temporal state is essential for calculating smooth changes in motion, such as acceleration and momentum. It also caches environmental data, like the valid vertical flight range, to avoid redundant world queries.

- **Thread Safety:** **This class is not thread-safe and must not be accessed concurrently.** It is designed to be owned and operated by a single NPC within the server's main entity update loop. All state modifications are performed synchronously during an entity's tick. Unsynchronized access from other threads will lead to state corruption, race conditions, and unpredictable physics calculations.

## API Surface
The public contract is primarily defined by its base class, MotionControllerBase, and is invoked by the NPC's core update logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeMove(ref, role, steering, dt, translation, accessor) | double | O(1) | **Core Method.** Calculates the final translation vector for the current tick based on steering inputs and internal physics state. This is a computationally intensive operation. |
| probeMove(ref, probeMoveData, accessor) | double | O(1) | Simulates a potential move along a given vector, returning the distance that can be traveled before a collision. Used for AI look-ahead and pathfinding validation. |
| getDesiredVerticalRange(ref, accessor) | VerticalRange | O(1) | Queries the world to determine the safe flight ceiling and floor at the NPC's current X/Z location, respecting configured height-over-ground parameters. |
| canAct(ref, accessor) | boolean | O(1) | Returns true if the NPC is in a state where it can actively fly (e.g., is in the air and not under a disabling status effect). |
| takeOff(ref, speed, accessor) | void | O(1) | A command to initiate flight, setting an initial upward velocity and speed. |
| onGround() | boolean | O(1) | Returns true if the internal PositionProbeAir determines the NPC is resting on a solid surface. |

## Integration Patterns

### Standard Usage
The MotionControllerFly is managed by the NPC's Role component. On each server tick, the AI system (e.g., a Behavior Tree) populates a Steering object with movement intentions. The Role then invokes the motion controller's update cycle, which calls computeMove to generate the final physics-aware translation.

```java
// Within an NPC's update loop
Steering steering = npc.getSteering(); // Populated by AI
MotionControllerFly controller = npc.getMotionController();
Vector3d translation = new Vector3d();

// The controller calculates the final translation vector for the tick
controller.computeMove(npc.getRef(), npc.getRole(), steering, dt, translation, accessor);

// The result is then applied to the NPC's transform
controller.executeMove(npc.getRef(), npc.getRole(), dt, translation, accessor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MotionControllerFly()`. The object is complex and must be configured through the builder pattern during NPC creation to load its physical properties correctly.
- **State Tampering:** Do not modify public state fields like lastVelocity or lastSpeed from outside the class. This will break the internal physics simulation and cause jerky, unpredictable movement.
- **Multiple Updates per Tick:** Calling computeMove or executeMove more than once per entity update tick will corrupt the time-dependent calculations (e.g., acceleration over dt) and lead to incorrect velocities.
- **Cross-Thread Access:** As stated, this component is not thread-safe. All interactions must originate from the owning entity's designated update thread.

## Data Pipeline
The MotionControllerFly is a key processing stage in the NPC movement data pipeline. It transforms abstract intent into concrete physical changes.

> Flow:
> AI Behavior Tree -> Steering Object (Desired Velocity) -> **MotionControllerFly.computeMove** -> Raw Translation Vector -> CollisionModule Query -> Collision-Adjusted Translation -> **MotionControllerFly.executeMove** -> TransformComponent Update

