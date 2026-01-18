---
description: Architectural reference for BodyMotionFindBase
---

# BodyMotionFindBase<T>

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Component

## Definition
```java
// Signature
public abstract class BodyMotionFindBase<T extends AStarBase> extends BodyMotionBase implements AStarEvaluator {
```

## Architecture & Concepts
BodyMotionFindBase is an abstract base class that serves as the brain for NPC pathfinding and movement. It orchestrates a sophisticated, hybrid motion strategy that combines long-range, deliberative pathfinding with short-range, reactive steering. This component acts as a state machine, guiding an NPC from a starting point to a destination through potentially complex environments.

Its core responsibility is to bridge the gap between a high-level goal (e.g., "move to location X") and the low-level steering commands required each tick to achieve it. It does this by managing two key sub-components:

1.  **AStar Pathfinding:** An instance of AStarBase (provided via the generic parameter T) is used for computationally intensive, grid-based pathfinding. This is invoked when direct movement is not possible or when a long-distance path is required. The path computation is time-sliced, processing a fixed number of nodes per tick (`nodesPerTick`) to avoid server performance degradation.

2.  **Path Following & Steering:** Once a path is computed by AStar, the internal PathFollower utility takes over. It translates the sequence of nodes into a smooth path, managing waypoints and generating continuous steering vectors. If no path is needed (i.e., the target is unobstructed), the component can fall back to a simpler, direct steering behavior.

The central logic resides within the `computeSteering` method, which functions as a large state machine. On every game tick, it evaluates the NPC's current situation and transitions between internal states such as COMPUTING, FOLLOWING, STEERING, BLOCKED, or AT_GOAL. This state is communicated externally via the MotionController's NavState.

### Lifecycle & Ownership
- **Creation:** Instantiated via its corresponding `BuilderBodyMotionFindBase` during the configuration of an NPC's behavior profile or Role. It is never created directly with `new`. The builder pattern allows for extensive configuration of its numerous parameters, such as pathfinding limits, throttling delays, and debug flags.
- **Scope:** The component is activated when an NPC's active behavior requires path-based movement. It persists as long as that behavior is active. Its lifecycle is tightly coupled to the parent Role that owns it.
- **Destruction:** The `deactivate` method is invoked when the behavior concludes or is interrupted. This method is critical for cleanup, as it clears any cached path data from the AStar instance and the PathFollower, preventing state leakage between behaviors. The object is then eligible for garbage collection.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains the current state of the pathfinding process, including the AStar instance, the active path in PathFollower, throttling timers, and waypoint progress. This internal state is modified on nearly every invocation of `computeSteering`.

- **Thread Safety:** **Not thread-safe.** This component is designed to be exclusively operated on by the main server game loop thread. All state modifications assume sequential, non-concurrent access within a single tick-based update cycle. Unsynchronized access from other threads will lead to state corruption, race conditions, and unpredictable NPC behavior.

## API Surface
The primary interaction surface is designed for the NPC behavior system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(N) / O(1) | The main update method. Executes the motion state machine. Complexity is O(N) where N is `nodesPerTick` during path computation, and O(1) otherwise. Returns true if motion should continue. |
| activate(...) | void | O(1) | Prepares the component for use by acquiring shared resources and initializing state. Must be called before the first `computeSteering` call. |
| deactivate(...) | void | O(1) | Cleans up internal state, primarily by clearing cached path data. |
| findBestPath(...) | void | - | **Abstract.** Subclasses must implement this to define fallback logic when a perfect path to the goal cannot be found. |
| startPathFinder(...) | boolean | O(N) | Initiates a new A* path computation. The complexity depends on the pathfinding implementation and can be significant. |
| continuePathFinder(...) | boolean | O(N) | Continues a time-sliced path computation, processing up to `nodesPerTick` nodes. |

## Integration Patterns

### Standard Usage
BodyMotionFindBase is not used directly but is extended by concrete implementations. A parent system, such as a Role, holds an instance of this component and calls `computeSteering` on each game tick to populate a Steering object.

```java
// Conceptual example within an NPC's update loop
// Assume 'motionComponent' is an instance of a BodyMotionFindBase subclass.

// The Steering object is an output parameter, filled by computeSteering.
Steering desiredSteering = new Steering();

// This is called every server tick.
boolean canContinueMotion = motionComponent.computeSteering(
    entityRef,
    npcRole,
    infoProvider,
    deltaTime,
    desiredSteering,
    componentAccessor
);

if (canContinueMotion) {
    // Apply the computed steering vectors to the NPC's physics/transform.
    applySteeringToEntity(entityRef, desiredSteering);
} else {
    // The component has reached its goal, failed, or aborted.
    // The behavior tree should transition to a new state.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate a subclass of BodyMotionFindBase directly. Always use the corresponding builder to ensure all parameters are correctly configured from asset files.
- **State Tampering:** Do not externally modify the state of the internal `aStar` or `pathFollower` objects. The component's state machine manages their lifecycles. External modification will break the pathfinding logic.
- **Concurrent Access:** Do not call `computeSteering` or other methods from any thread other than the main server tick thread. This will cause severe and difficult-to-debug concurrency issues.
- **Ignoring Return Value:** The boolean return value of `computeSteering` is a critical signal indicating whether the motion task is complete (goal reached, aborted, blocked). Ignoring it can cause an NPC to become stuck in a behavior that should have already terminated.

## Data Pipeline
The component transforms a high-level goal into low-level movement commands by processing data through a distinct pipeline on each tick.

> Flow:
> Game Tick → Role Update → **BodyMotionFindBase.computeSteering()** → Read (TransformComponent, MotionController) → State Machine Logic → AStar.computePath() → PathFollower.setPath() → PathFollower.executePath() → Write (Steering object) → Physics System → Update (TransformComponent)

