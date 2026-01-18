---
description: Architectural reference for PathFollower
---

# PathFollower

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Transient

## Definition
```java
// Signature
public class PathFollower {
```

## Architecture & Concepts
The PathFollower is a stateful component responsible for translating an abstract navigation path into concrete steering commands for an NPC. It does not generate paths; rather, it consumes a pre-computed linked list of IWaypoint objects provided by a higher-level pathfinding system, such as an A* implementation.

Its core architectural function is to act as a state machine that guides an entity along a sequence of discrete points. On each game tick, it evaluates the entity's proximity to the current waypoint and determines whether to advance to the next. It then calculates a desired velocity vector, incorporating advanced behaviors like path smoothing and overshoot correction, which is fed into the NPC's MotionController.

This class is a critical bridge between high-level AI planning (the path) and low-level physics-driven movement (the MotionController). It is designed for single-entity ownership and is not a shared or global service.

### Lifecycle & Ownership
- **Creation:** A PathFollower is instantiated by an NPC's behavior or task controller when a navigation goal is initiated. It is not managed by a central registry or service locator.
- **Scope:** The object's lifetime is tied directly to a single navigation task. It persists as long as the NPC is actively following the assigned path.
- **Destruction:** The instance is eligible for garbage collection once the NPC reaches its destination, the navigation task is cancelled, or the owning NPC is despawned. The `clearPath` method should be called to release references to the waypoint chain, though typically the entire object is discarded.

## Internal State & Concurrency
- **State:** The PathFollower is highly mutable. Its primary state includes a reference to the `currentWaypoint`, the position of the last waypoint, and numerous configuration parameters like `waypointRadius` and `rejectionWeight`. It also maintains several internal Vector3d instances to avoid heap allocations during its frequent calculations.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be exclusively owned and operated by a single NPC entity within the main server game loop. All public methods modify internal state without any synchronization primitives. Concurrent access from multiple threads will result in state corruption and undefined behavior.

## API Surface
This section details the primary public contract for the PathFollower. Trivial setters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setPath(IWaypoint, Vector3d) | void | O(1) | Initializes the follower with a new path. This is the primary entry point for starting a navigation task. |
| clearPath() | void | O(1) | Resets the internal state, clearing the current path. Essential for cancelling navigation. |
| updateCurrentTarget(Vector3d, MotionController) | boolean | O(1) | Advances the internal state to the next waypoint if the current one has been reached. Must be called each tick before `executePath`. |
| executePath(Vector3d, MotionController, Steering) | void | O(1) | Calculates the final steering vector based on the current position and target waypoint. Populates the provided Steering object. |
| smoothPath(...) | void | O(N) | Performs a series of line-of-sight checks to "cut corners" and skip unnecessary waypoints. N is bounded by `pathSmoothing`. |
| freezeWaypoint() | boolean | O(1) | Locks onto the final waypoint in a path, ensuring the NPC stops precisely at the destination. |

## Integration Patterns

### Standard Usage
The PathFollower is intended to be used within an NPC's per-tick update logic. The typical sequence involves updating the target, potentially smoothing the path, and then executing the path to generate a steering command.

```java
// Within an NPC's update method
// Assumes 'this.pathFollower' is a member field of the NPC's controller

// 1. Check if the current waypoint is reached and advance if necessary
boolean hasPath = this.pathFollower.updateCurrentTarget(entity.getPosition(), motionController);

if (!hasPath) {
    // Path is complete or was cleared, stop moving.
    return;
}

// 2. (Optional) If path smoothing is enabled, attempt to cut corners
if (this.pathFollower.shouldSmoothPath()) {
    this.pathFollower.smoothPath(...);
}

// 3. Calculate the steering vector for the current frame
Steering desiredSteering = new Steering();
this.pathFollower.executePath(entity.getPosition(), motionController, desiredSteering);

// 4. Apply the steering to the NPC's motion controller
motionController.setSteering(desiredSteering);
```

### Anti-Patterns (Do NOT do this)
- **State Tampering:** Do not attempt to modify the `currentWaypoint` field directly. Always manage the path via the `setPath` and `clearPath` methods.
- **Update Order Violation:** Calling `executePath` without first calling `updateCurrentTarget` in the same tick will cause the NPC to fixate on a single waypoint and never advance through the path.
- **Concurrent Modification:** Do not access a PathFollower instance from any thread other than the main server thread that owns the associated NPC.
- **Path Re-use:** Do not attempt to share a single IWaypoint chain across multiple PathFollower instances. The path structure is treated as a consumable resource.

## Data Pipeline
The PathFollower is a processing stage in the NPC movement pipeline. It transforms a discrete list of world coordinates into a continuous steering signal.

> Flow:
> Pathfinding System (A*) -> IWaypoint Linked List -> **PathFollower.setPath()** -> Game Tick -> **PathFollower.updateCurrentTarget()** -> **PathFollower.executePath()** -> Steering Vector -> MotionController -> Final Entity Velocity

