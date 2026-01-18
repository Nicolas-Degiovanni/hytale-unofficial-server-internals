---
description: Architectural reference for AStarWithTarget
---

# AStarWithTarget

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Transient State Object

## Definition
```java
// Signature
public class AStarWithTarget extends AStarBase {
```

## Architecture & Concepts
AStarWithTarget is a concrete implementation of the A* pathfinding algorithm, specializing the generic AStarBase framework. Its primary function within the server architecture is to compute a complete or partial navigation path for a non-player character (NPC) towards a fixed coordinate in the world.

This class acts as a stateful, single-use calculator for a navigation query. It is a critical component of the server's AI system, translating a high-level goal (e.g., "move to position X,Y,Z") into a low-level, traversable series of waypoints.

The key architectural feature of this class is its resilience to unreachable targets. If a direct path to the goal is obstructed or does not exist, the `findClosestPath` method allows the system to recover by building a path to the most promising reachable node. This design prevents NPCs from freezing or behaving erratically when their target is inside a wall or across an impassable obstacle, enabling more robust and believable AI behavior.

## Lifecycle & Ownership
- **Creation:** An AStarWithTarget instance is created on-demand by a higher-level AI system, such as an entity's behavior tree or a central navigation service, whenever a path to a static point is required. It is not a persistent or shared service.

- **Scope:** The object's lifetime is strictly bound to a single pathfinding computation. It is instantiated, configured via `initComputePath`, used to run the pathfinding steps, and then its result is extracted. After this, the object holds stale data and is intended to be discarded.

- **Destruction:** The object is managed by the Java garbage collector. There is no explicit cleanup or `destroy` method. It becomes eligible for collection as soon as the calling system has retrieved the computed path and drops its reference to the instance.

## Internal State & Concurrency
- **State:** AStarWithTarget is a highly mutable, stateful object. It encapsulates all intermediate data for a single A* calculation, including the open and closed sets of nodes (inherited from AStarBase), the final computed path, and its own specific `targetPosition`. The object's state is only valid for the duration of one computation.

- **Thread Safety:** This class is **not thread-safe**. All internal state is accessed without locks or other synchronization primitives. It is designed for synchronous, single-threaded execution within an entity's update cycle or a dedicated AI thread.

> **Warning: Concurrency Hazard**
>
> Concurrent modification of an AStarWithTarget instance from multiple threads will result in a corrupted internal state, leading to invalid path results, infinite loops, and server instability. All pathfinding operations using this class must be strictly serialized.

## API Surface
The public API provides the necessary hooks to initialize, execute, and finalize a pathfinding query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initComputePath(...) | AStarBase.Progress | O(1) | Configures the pathfinder for a new computation. This must be called before any other processing. It populates all required context, including start/end points, world data, and entity motion constraints. |
| findClosestPath() | float | O(N) | Scans all visited nodes to find the one with the lowest estimated cost to the target. It then builds the final path from that node. N is the number of nodes explored by the algorithm. Returns the cost, or Float.MAX_VALUE if no path is possible. |
| createDebugHelper(...) | AStarDebugWithTarget | O(1) | Instantiates a companion object used for logging and debugging the pathfinding process. |
| getTargetPosition() | Vector3d | O(1) | Returns a reference to the target position vector. |

## Integration Patterns

### Standard Usage
AStarWithTarget is used as part of a larger AI behavior. The controlling system is responsible for supplying all environmental and entity-specific context.

```java
// Conceptual example within an AI Behavior class

// 1. Obtain a pathfinder instance
AStarWithTarget pathfinder = new AStarWithTarget();

// 2. Initialize with world context and query parameters
pathfinder.initComputePath(
    entityStoreRef,
    npc.getPosition(),
    goalPosition,
    pathingEvaluator,
    npc.getMotionController(),
    npc.getProbeMoveData(),
    nodePoolProvider,
    componentAccessor
);

// 3. Execute the A* search loop (methods on AStarBase)
while (pathfinder.computeStep() == AStarBase.Progress.COMPUTING) {
    // This loop can be time-sliced across multiple ticks
}

// 4. Finalize the path, gracefully handling unreachable targets
pathfinder.findClosestPath();
Path result = pathfinder.getPath();

// 5. Assign the result to the NPC
npc.getNavigator().setPath(result);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not attempt to reuse an AStarWithTarget instance for a new pathfinding query without calling `initComputePath` again. The object's internal state is specific to the previous computation and will produce incorrect results.

- **Long-Term References:** Do not store instances of AStarWithTarget in long-lived components. They are lightweight, transient objects designed to be created and discarded for each path request.

- **Ignoring `findClosestPath`:** Relying solely on the path being complete can cause issues. If the target is unreachable, the path will be null until `findClosestPath` is called to construct the best partial path available.

## Data Pipeline
The class transforms a high-level navigation query into a low-level waypoint list by processing world data.

> Flow:
> AI Behavior Request (Start, End) -> **AStarWithTarget.initComputePath** -> A* Search Loop (evaluating world collision via `AStarEvaluator`) -> **AStarWithTarget.findClosestPath** -> Internal `Path` object -> NPC `MotionController`

