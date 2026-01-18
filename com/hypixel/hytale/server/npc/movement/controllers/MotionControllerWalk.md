---
description: Architectural reference for MotionControllerWalk
---

# MotionControllerWalk

**Package:** com.hypixel.hytale.server.npc.movement.controllers
**Type:** Transient

## Definition
```java
// Signature
public class MotionControllerWalk extends MotionControllerBase {
```

## Architecture & Concepts
The MotionControllerWalk is a concrete implementation of the MotionControllerBase, providing the fundamental physics simulation for terrestrial, non-flying NPCs. It is the primary system responsible for translating high-level AI steering intentions into low-level, collision-aware positional updates within the game world.

This controller operates as a stateful physics and movement kernel for an individual NPC. It manages a complex internal state machine governed by the MotionKind enum, which dictates behavior for states such as STANDING, MOVING, ASCENDING (climbing or jumping), and DESCENDING (falling or controlled descent).

Its core architectural function is to act as the bridge between the abstract pathfinding and behavior systems and the concrete world geometry. It consumes a Steering object, which represents the desired direction and speed, and produces a valid, collision-free translation vector that is applied to the NPC's TransformComponent each tick. This involves continuous and intensive interaction with the CollisionModule and direct queries against the world's block and fluid data.

Key responsibilities include:
-   **Ground Detection:** Determining if the NPC is on the ground (`onGround`) or in a fluid (`inWater`).
-   **Gravity and Falling:** Applying gravity and calculating fall speed when the NPC is airborne.
-   **Collision Resolution:** Executing moves and resolving collisions with world geometry, including sliding along walls.
-   **Traversal Logic:** Implementing advanced traversal maneuvers like step-climbing over single-block obstacles and multi-block jumps.
-   **State Synchronization:** Updating the `MovementStates` component, which drives the NPC's animations based on its physical actions (walking, running, jumping, falling).

## Lifecycle & Ownership
-   **Creation:** MotionControllerWalk is instantiated via its corresponding builder, BuilderMotionControllerWalk. This process is typically driven by the NPC asset loading system when an NPC is first spawned into the world. The builder pattern ensures that the controller is configured with dozens of specific physics parameters (e.g., jumpHeight, maxClimbHeight, acceleration) defined in the NPC's asset files.
-   **Scope:** The lifecycle of a MotionControllerWalk instance is tightly coupled to the NPCEntity it controls. It persists for the entire duration that the NPC exists in the world. Each walking NPC has its own unique instance, maintaining its specific physical state.
-   **Destruction:** The object is eligible for garbage collection when its owning NPCEntity is despawned and all references to it are released. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains a large set of fields representing the NPC's instantaneous physical state, including `onGround`, `inWater`, `moveSpeed`, `fallSpeed`, and the current `MotionKind`. It also caches numerous configuration parameters from its builder. To optimize performance and reduce memory allocation, it heavily utilizes temporary, reusable Vector3d objects (e.g., `tmpClimbPosition`, `tmpMovePosition`) for intermediate calculations within the update loop.

-   **Thread Safety:** **This class is not thread-safe and must be considered thread-hostile.** It is designed to be exclusively accessed and manipulated by the single thread responsible for its owner NPC's updates (typically the main server thread or a dedicated world thread). All methods that modify state do so without any synchronization primitives. Concurrent access would result in state corruption, race conditions, and unpredictable physics behavior.

## API Surface
The public contract is primarily defined by overrides of MotionControllerBase methods, which are called in a specific sequence by the NPC update loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeMove(ref, role, steering, dt, translation, accessor) | double | O(N) | Core logic tick. Calculates the desired translation based on steering input and internal state. Complexity depends on the current MotionKind. |
| executeMove(ref, role, dt, translation, accessor) | double | O(M) | Applies the computed translation, performs collision detection, and updates the final position. Complexity depends on the number of potential block collisions (M). |
| postReadPosition(ref, accessor) | void | O(W*D) | Synchronizes internal state with the world. Checks blocks under the NPC's bounding box to update `onGround` and `inWater` status. Complexity depends on the width (W) and depth (D) of the NPC's footing. |
| probeMove(ref, probeData, accessor) | double | O(L*M) | Simulates a move along a path without modifying state. Used by pathfinding to validate traversability. Complexity depends on path length (L) and potential collisions (M). |
| canAct(ref, accessor) | boolean | O(1) | Returns true if the NPC is in a state to perform actions, primarily checking if it is on solid ground. |
| translateToAccessiblePosition(position, ...) | boolean | O(H) | Finds the highest solid block at an X/Z coordinate within a vertical range. Used for safe spawning. Complexity depends on the height (H) of the search space. |

## Integration Patterns

### Standard Usage
The MotionControllerWalk is not intended for direct use by most developers. It is driven by the server's internal NPC processing loop. The standard operational sequence for a single game tick is critical and strictly ordered.

```java
// Simplified representation of the server's NPC update loop

// 1. Read current position from the Entity's TransformComponent
controller.readPosition(ref, accessor);

// 2. Synchronize with world state (check for ground, water, etc.)
controller.postReadPosition(ref, accessor);

// 3. Get steering input from AI/Pathfinding
Steering steering = npc.getSteering();

// 4. Calculate the desired movement vector for this frame
Vector3d translation = new Vector3d();
controller.computeMove(ref, role, steering, dt, translation, accessor);

// 5. Apply movement, handle collisions, and update internal position
controller.executeMove(ref, role, dt, translation, accessor);

// 6. Write the final, validated position back to the Entity's TransformComponent
controller.writePosition(ref, dt, accessor);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new MotionControllerWalk()`. The controller has extensive configuration that must be injected via the `BuilderMotionControllerWalk`. Failure to use the builder will result in a controller with default or null parameters, leading to undefined behavior.
-   **State Tampering:** Do not externally modify the controller's state fields, such as `onGround` or `moveSpeed`. The internal state is tightly coupled and managed by the physics simulation. External changes will break the simulation's integrity.
-   **Incorrect Call Order:** The sequence of `postReadPosition`, `computeMove`, and `executeMove` is mandatory. Executing a move before checking the ground state with `postReadPosition` will cause the controller to operate on stale data from the previous tick, leading to incorrect physics calculations.
-   **Multi-threaded Access:** As stated, never call methods on this class from any thread other than the one designated for entity updates. This will cause severe and difficult-to-debug race conditions.

## Data Pipeline
The MotionControllerWalk processes data in a linear flow each tick to transform an AI's intent into a physical change in the world.

> Flow:
> AI Steering Intention -> `computeMove` -> Desired Translation Vector -> `executeMove` -> Collision Detection -> Resolved Translation Vector -> Internal Position Update -> `writePosition` -> Entity TransformComponent Update

