---
description: Architectural reference for BodyMotionFindWithTarget
---

# BodyMotionFindWithTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Stateful Component

## Definition
```java
// Signature
public abstract class BodyMotionFindWithTarget extends BodyMotionFindBase<AStarWithTarget> {
```

## Architecture & Concepts
BodyMotionFindWithTarget is a foundational, abstract component within the server-side NPC AI framework, responsible for orchestrating an entity's movement towards a dynamic target. It serves as a specialized layer on top of BodyMotionFindBase, introducing the logic necessary to track, pathfind to, and react to a moving entity or location.

This component is not a standalone system; it functions as a strategic module activated by an NPC's current **Role**. For example, a "ChasePlayer" role would activate an implementation of this component to govern its movement.

Its core architectural responsibilities include:

*   **Target State Caching:** It captures and caches the target's position and bounding box from an InfoProvider at the beginning of a logic tick.
*   **Path Recomputation Strategy:** Its primary function is to determine *when* a new path is required. This is a critical performance optimization. Recomputation is triggered by several conditions:
    1.  The target has moved beyond a configured distance threshold (minMoveDistanceRecompute).
    2.  The target has moved outside a pre-defined view cone, preventing the NPC from appearing unintelligent if the target circles behind it.
    3.  The existing path is invalidated for other reasons (e.g., blocked).
*   **Target Accessibility Projection:** A key feature is its ability to differentiate between a target's actual position and its nearest *accessible* position. The method getLastAccessibleTargetPosition translates a target's raw coordinates (which may be inside a wall or in the air) to a valid, reachable point on the navigation mesh. This prevents the pathfinder from failing on trivial environmental blockages.
*   **Pathfinding Throttling:** To prevent excessive pathfinding requests for slow-moving or stationary targets, it implements a deferral mechanism. If a target has barely moved, it can delay recomputation, entering a `waitForTargetMovement` state.

This class embodies the "sense-think-act" AI loop. It "senses" the target's position via InfoProvider, "thinks" by evaluating recomputation and accessibility rules, and enables the "act" phase by initiating an AStarWithTarget pathfinding job.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the new keyword. They are instantiated by the NPC asset loading system via a corresponding builder class, BuilderBodyMotionFindWithTarget. This process typically occurs once when the server loads NPC definitions.
-   **Scope:** The component's lifecycle is tightly coupled to the **Role** that owns it. It is inert until its `activate` method is called by the Role when this specific movement strategy is required. Its state persists only as long as the motion strategy remains active.
-   **Destruction:** The object is eligible for garbage collection when its parent Role is deactivated or the owning NPC entity is unloaded. The `activate` method serves as a re-initialization point, resetting its internal state for subsequent uses.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains numerous fields representing the last known state of its target (lastTargetPosition, targetBoundingBox), its own pathing calculations (lastPathedPosition), and behavioral flags (waitForTargetMovement, haveValidTargetPosition). This state is essential for making decisions that span multiple server ticks.

-   **Thread Safety:** **This class is not thread-safe and must only be accessed from the main server thread.** It performs no internal locking and its state is designed to be mutated sequentially within a single NPC's update tick. Concurrent access from other threads will lead to state corruption, race conditions, and unpredictable NPC behavior.

## API Surface
The public API constitutes the contract with the NPC's MotionController and Role systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(ref, role, accessor) | void | O(1) | Resets the component's internal state. Called when the owning Role makes this motion strategy active. |
| canComputeMotion(ref, role, info, accessor) | boolean | O(1) | Checks if a valid target exists via the InfoProvider and caches its state for the current tick. Must be called before other update methods. |
| mustRecomputePath(controller) | boolean | O(1) | Evaluates distance and view-cone heuristics to determine if the current path is stale and needs recalculation. |
| shouldDeferPathComputation(controller, pos, accessor) | boolean | O(1) | Implements throttling logic. Returns true if pathfinding should be skipped because the target has not moved a significant distance. |
| startComputePath(ref, role, controller, pos, accessor) | AStarBase.Progress | O(1) | Initiates an A* pathfinding job. The operation itself is asynchronous; this method returns immediately with a progress handle. |
| onNoPathFound(controller) | void | O(1) | A callback invoked by the pathfinding system when no valid path to the target can be found. Sets the internal state to wait for target movement. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in code. Instead, its behavior is defined declaratively in NPC asset files, which are then parsed by a builder to create an instance. The game's AI engine then invokes its methods in a prescribed order during the NPC update cycle.

*Conceptual asset definition:*
```yaml
# Example NPC asset file (conceptual)
role: "AggressiveMelee"
motion:
  type: "FindWithTarget"
  params:
    minMoveDistanceRecompute: 4.0
    recomputeConeAngle: 90.0
    adjustRangeByHitboxSize: true
```

*Conceptual engine usage:*
```java
// This logic resides deep within the NPC Role/MotionController systems.
// It is NOT typical user code.

// During an NPC's update tick:
if (motion instanceof BodyMotionFindWithTarget) {
    BodyMotionFindWithTarget findMotion = (BodyMotionFindWithTarget) motion;

    if (findMotion.canComputeMotion(...)) {
        if (findMotion.mustRecomputePath(...)) {
            if (!findMotion.shouldDeferPathComputation(...)) {
                findMotion.startComputePath(...);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BodyMotionFindWithTarget(...)`. The object will be unconfigured and will bypass the crucial builder pattern, leading to NullPointerExceptions and incorrect behavior.
-   **Incorrect Invocation Order:** The public methods are designed to be called in a specific sequence during a single game tick (e.g., `canComputeMotion` before `mustRecomputePath`). Calling them out of order will cause decisions to be made based on stale data from a previous tick.
-   **State Manipulation:** Do not externally modify the public fields of this class. Its internal state is carefully managed by its own methods.
-   **Multi-threaded Access:** Never share an instance across threads or call its methods from any thread other than the primary server tick thread for that NPC.

## Data Pipeline
This component acts as a decision-making filter in the NPC movement data pipeline, translating high-level target information into a concrete pathfinding request.

> Flow:
> NPC Sensor System -> InfoProvider -> **BodyMotionFindWithTarget** (Evaluates Target) -> AStarWithTarget (Path Calculation) -> MotionController (Path Following) -> TransformComponent (Entity Position Update)

