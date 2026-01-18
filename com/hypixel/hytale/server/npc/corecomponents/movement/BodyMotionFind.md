---
description: Architectural reference for BodyMotionFind
---

# BodyMotionFind

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionFind extends BodyMotionFindWithTarget {
```

## Architecture & Concepts
The BodyMotionFind class is a concrete implementation of a movement strategy within the server-side NPC AI framework. Its primary function is to generate the necessary motion and steering data for an NPC to navigate towards and arrive at a specific target entity or coordinate.

This component operates as a high-level "goal-oriented" motion planner. It does not directly manipulate entity physics. Instead, it computes a desired state—a `Steering` object—which is then consumed by a lower-level `MotionController` to be translated into physical forces and velocity changes.

Architecturally, BodyMotionFind employs a hybrid navigation model to balance performance and behavior fidelity:

1.  **Long-Range Pathfinding:** For distant targets, it integrates with the AStar navigation system (`AStarBase`). It provides the necessary heuristics (`estimateToGoal`) and goal-completion checks (`isGoalReached`) to guide the pathfinder.
2.  **Short-Range Steering:** Once the NPC is within a configurable radius of the target (`switchToSteeringDistance`), the system transitions from path-following to a direct, reactive pursuit. This is handled by an internal `SteeringForcePursue` instance, which calculates a direct interception vector. This switch avoids the computational overhead and potential path-correction jitter of AStar at close quarters, resulting in more natural-looking arrival behavior.

This class is fundamental for any AI behavior that requires an NPC to find and move to a dynamic or static target, such as pursuing a player, moving to a patrol point, or approaching an interactive object.

### Lifecycle & Ownership
- **Creation:** BodyMotionFind is not intended for direct instantiation. It is constructed via its corresponding builder, `BuilderBodyMotionFind`, as part of the NPC's behavior configuration. This typically occurs when a behavior tree or state machine activates a goal that requires this type of movement.
- **Scope:** The object's lifetime is tied directly to the AI goal it serves. An instance of BodyMotionFind exists only as long as the NPC is actively trying to "find" its assigned target. Once the goal is achieved, aborted, or superseded, this object is discarded.
- **Destruction:** Cleanup is managed by the Java garbage collector. There are no manual tear-down or `close` methods, as the object holds no unmanaged resources.

## Internal State & Concurrency
- **State:** This class is highly stateful. It caches numerous configuration parameters from its builder, such as stopping distances, abort thresholds, and height difference tolerances. It also maintains a mutable, temporary `Vector3d` (`tempDirectionVector`) to prevent object allocation during frequent calculations within the game tick.
- **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** It is designed to be invoked exclusively from the main server thread during the NPC update cycle. The use of a shared, mutable `tempDirectionVector` for internal calculations is a clear indicator of this design assumption; concurrent access would lead to data corruption and unpredictable NPC movement.

## API Surface
The primary API consists of overridden methods that plug into the NPC's motion and navigation pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canComputeMotion(...) | boolean | O(1) | Predicate to determine if this motion strategy can execute. Checks abort distance and bounding box validity. |
| computeSteering(...) | boolean | O(1) | Calculates the final steering vector using the internal `SteeringForcePursue` instance. Only active during the short-range phase. |
| isGoalReached(...) | boolean | O(1) | The termination condition for the movement. Checks distance, height difference, and line-of-sight reachability. |
| canSwitchToSteering(...) | boolean | O(N) | Determines if the system should transition from AStar pathfinding to direct steering. May involve a world probe via `MotionController.probeMove`. |
| estimateToGoal(...) | float | O(1) | Provides the heuristic (straight-line distance) for the AStar pathfinder. |
| findBestPath(...) | void | O(1) | A configuration callback that instructs the AStar system on how to select the optimal path from its open set. |

## Integration Patterns

### Standard Usage
This component is not used in isolation. It is configured and managed by an NPC's `Role`, which orchestrates the various AI components. The game engine invokes its methods during the NPC update tick.

```java
// Conceptual example within an NPC's behavior update

// The Role would have already configured and activated BodyMotionFind
// based on a high-level goal like "PURSUE_PLAYER".

// During the server tick for this NPC:
MotionController controller = npc.getRole().getActiveMotionController();
BodyMotionFind currentMotion = (BodyMotionFind) controller.getActiveMotionComponent();
Steering desiredSteering = new Steering();

if (currentMotion.canComputeMotion(npcRef, npc.getRole(), ...)) {
    // The engine will internally call computeSteering or use AStar
    // based on the logic within BodyMotionFind.
    // The result is a populated Steering object.
    controller.applySteering(npcRef, desiredSteering, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BodyMotionFind()`. The object is complex and requires numerous parameters that are correctly supplied only by its designated builder, `BuilderBodyMotionFind`. Bypassing the builder will result in a misconfigured or non-functional component.
- **State Reuse:** Do not attempt to cache and reuse a BodyMotionFind instance for different goals or different NPCs. Its internal state is tied to a specific target and configuration.
- **Concurrent Modification:** Do not access or call methods on this object from any thread other than the main server tick thread. This will cause race conditions, particularly with the `tempDirectionVector` state.

## Data Pipeline
BodyMotionFind acts as a translator, converting a high-level goal (a target position) into a low-level steering command.

> Flow:
> AI Goal (Target Position) -> **BodyMotionFind** -> AStar Path / SteeringForcePursue -> `Steering` Object -> `MotionController` -> Entity Physics Update

