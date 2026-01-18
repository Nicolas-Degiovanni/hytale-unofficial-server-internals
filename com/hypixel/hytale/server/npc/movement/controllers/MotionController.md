---
description: Architectural reference for the MotionController interface, the core contract for NPC movement logic.
---

# MotionController

**Package:** `com.hypixel.hytale.server.npc.movement.controllers`
**Type:** Contract Interface

## Definition
```java
// Signature
public interface MotionController {
    // ... public abstract methods
}
```

## Architecture & Concepts

The MotionController is a foundational interface within the server-side NPC AI framework. It defines the contract for all concrete NPC movement strategies, acting as the bridge between high-level behavioral logic (the `Role`) and the low-level physics and world simulation.

This interface embodies the **Strategy Pattern**. An NPC's `Role` component determines *what* the NPC wants to do (e.g., "seek target", "wander area"), while the assigned `MotionController` implementation determines *how* the NPC physically accomplishes it (e.g., walking, flying, swimming). This decoupling allows for modular and swappable movement logic without altering the NPC's core behavioral programming.

A `MotionController` is responsible for:
1.  **Steering & Locomotion:** Translating abstract steering intentions into concrete forces and velocity changes.
2.  **Environmental Probing:** Querying the world to determine valid moves, check for obstacles, and assess terrain accessibility.
3.  **State Synchronization:** Updating the entity's `MovementStatesComponent` to reflect its physical actions (e.g., jumping, falling), which is critical for client-side animation.
4.  **Physics Interaction:** Responding to external forces like knockback and gravity.

Implementations of this interface are highly specialized. For example, a `WalkingMotionController` will have logic for gravity, ground friction, and step height, whereas a `FlyingMotionController` will manage altitude, air friction, and three-dimensional pathing.

## Lifecycle & Ownership

The lifecycle of a `MotionController` implementation is tightly bound to the NPC entity it controls.

-   **Creation:** A concrete `MotionController` is instantiated and assigned to an NPC's `Role` component when the NPC is spawned or when its fundamental mode of transport changes. The specific implementation is chosen based on the NPC's type and current environment (e.g., entering water may swap a walking controller for a swimming one).
-   **Scope:** The controller object persists as long as the NPC is using that specific movement strategy. It is a stateful object, holding data about the NPC's current momentum, navigation state, and environmental context.
-   **Destruction:** The object is eligible for garbage collection when the NPC is despawned or when its `Role` switches to a different `MotionController` implementation. The `deactivate` method serves as the primary cleanup hook.

## Internal State & Concurrency

-   **State:** All implementations of `MotionController` are expected to be **highly mutable and stateful**. They must track internal forces, navigation progress (`NavState`), environmental flags (e.g., `onGround`, `inWater`), and temporary movement constraints. This state is updated every server tick.
-   **Thread Safety:** `MotionController` implementations are **not thread-safe** and must not be treated as such. All methods on this interface are designed to be called exclusively from the main server tick thread for a given entity. Concurrent access from other threads will lead to race conditions, corrupted physics state, and unpredictable server behavior.

## API Surface

The public API defines the complete contract for controlling an NPC's physical presence in the world. The most critical methods are highlighted below.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steer(...) | double | O(N) | The core update method. Calculates and applies forces based on desired steering. Complexity depends on environmental queries. |
| probeMove(...) | double | O(N) | Simulates a potential move to test for collisions and accessibility. A critical tool for pathfinding and obstacle avoidance. |
| updateMovementState(...) | void | O(1) | Updates the entity's shared `MovementStatesComponent` based on the controller's internal state. |
| addForce(...) | void | O(1) | Applies an external, instantaneous force to the NPC, such as from an explosion. |
| forceVelocity(...) | void | O(1) | Overwrites the NPC's velocity directly. Used for effects like scripted movements or stuns. |
| canAct(...) | boolean | O(1) | A predicate indicating if the controller is in a state that allows for new actions (e.g., not stunned or mid-jump). |
| setNavState(...) | void | O(1) | Sets the high-level navigation state, informing the controller of its pathfinding status (e.g., following path, stuck). |

## Integration Patterns

### Standard Usage

The `MotionController` is almost never interacted with directly. Instead, it is managed by the NPC's `Role` component, which is the primary driver of AI behavior. The `Role` calculates a desired `Steering` output and passes it to the controller during the entity's update tick.

```java
// Simplified example from within an NPC's Role update logic
Steering desiredSteering = calculatePathfindingSteering(targetPosition);
Steering actualSteering = new Steering(); // Output parameter

// Delegate the "how" of movement to the controller
// The controller will update the entity's velocity and transform
motionController.steer(
    entityRef,
    this, // The Role
    desiredSteering,
    actualSteering,
    deltaTime,
    componentAccessor
);

// Update animation state for clients
motionController.updateMovementState(...);
```

### Anti-Patterns (Do NOT do this)

-   **External State Management:** Do not attempt to manage an NPC's velocity or position directly if it has an active `MotionController`. The controller is the sole authority on the entity's movement. Bypassing it by writing directly to the `Velocity` or `TransformComponent` will be overwritten on the next tick and will break its internal state. Use `addForce` or `forceVelocity` instead.
-   **Ignoring `canAct`:** Do not issue new movement commands from a `Role` without first checking `motionController.canAct()`. This can interrupt uninterruptible actions (like a scripted jump) and lead to inconsistent behavior.
-   **Cross-Thread Access:** Never cache a `MotionController` instance and access it from an asynchronous task or different thread. All interactions must be marshaled back to the main server thread.

## Data Pipeline

The `MotionController` is a central processor in the NPC update loop, transforming high-level intent into low-level physics state.

> Flow:
> AI `Role` (Behavior Tree, Goal System) -> `Steering` object (desired direction/speed) -> **`MotionController.steer()`** -> Physics Calculations (forces, collisions) -> `Velocity` Component Update -> `TransformComponent` Update -> World Simulation Tick

