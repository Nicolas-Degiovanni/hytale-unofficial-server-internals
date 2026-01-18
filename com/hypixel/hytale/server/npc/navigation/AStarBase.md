---
description: Architectural reference for AStarBase
---

# AStarBase

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Transient

## Definition
```java
// Signature
public class AStarBase {
```

## Architecture & Concepts
The **AStarBase** class is the foundational engine for the A* pathfinding algorithm used by server-side NPCs. It is not a generic, static utility; rather, it is a stateful, single-use object designed to manage the lifecycle of one specific pathfinding query.

Its core responsibility is to execute the A* search algorithm in a time-sliced manner, preventing any single pathfinding request from monopolizing server tick time. The design decouples the generic search algorithm from game-specific logic through two key abstractions:
- **MotionController:** Provides the "physics" of movement. It is used to probe the world and determine if a move between two points is valid and what the cost of that move is. This allows the same A* engine to be used for NPCs that walk, fly, or swim.
- **AStarEvaluator:** Defines the goal of the search. It provides the heuristic for estimating the distance to the target and determines if a given node satisfies the completion criteria.

The search space is a discretized grid based on half-block increments, which provides a higher resolution than block-based pathfinding. World positions are converted to and from a packed long index for efficient lookup in the visited nodes map.

## Lifecycle & Ownership
- **Creation:** An instance of **AStarBase** is created by a higher-level navigation or AI behavior system when a path is required. The object is inert until **initComputePath** is called.
- **Scope:** The object's lifetime is strictly bound to a single pathfinding operation. It persists from the invocation of **initComputePath** until a terminal state is reached (e.g., **ACCOMPLISHED**, **TERMINATED**) or it is manually reset via **clearPath**.
- **Destruction:** The object is eligible for garbage collection once the caller has retrieved the path result and discarded its reference. The **clearPath** method is critical for releasing **AStarNode** objects back to the shared **AStarNodePool**, preventing memory leaks. Failure to call **clearPath** on abandoned or completed searches will exhaust the node pool.

## Internal State & Concurrency
- **State:** **AStarBase** is highly mutable. Its primary state consists of the **openNodes** list (the search frontier) and the **visitedBlocks** map (the closed set). These collections grow and change as the search progresses. It also maintains configuration parameters like **maxPathLength** and caches search direction vectors.

- **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread, typically the main server thread for the world in which the search is being performed. All internal collections are unsynchronized. Concurrent access will lead to unpredictable behavior, race conditions, and likely a **ConcurrentModificationException**. All pathfinding logic for an entity must be serialized.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initComputePath(...) | AStarBase.Progress | O(k) | Initializes a new pathfinding query. Clears previous state and sets up the starting node(s) by probing the immediate area. k is the number of initial probe points. |
| computePath(...) | AStarBase.Progress | O(N) | Processes up to N nodes from the open set. This is the main workhorse method, designed to be called incrementally across multiple ticks. |
| getPath() | AStarNode | O(1) | Returns the head of the resulting path, which is a linked list of **AStarNode** objects. Returns null if no path has been computed. |
| clearPath() | void | O(V) | Resets the internal state and deallocates all nodes in the **visitedBlocks** map back to the node pool. V is the number of visited nodes. |
| buildBestPath(...) | AStarNode | O(V) | If a goal was not reached, this method can be used to construct a path to the "best" node found so far, based on a provided weight function. |

## Integration Patterns

### Standard Usage
**AStarBase** is designed for an iterative, non-blocking execution pattern. A controlling system (e.g., an AI behavior tree) initializes the search and then calls **computePath** with a fixed budget of nodes per tick until a result is available.

```java
// Standard iterative pathfinding loop
AStarBase pathfinder = new AStarBase();

// On first tick, initialize the search
pathfinder.initComputePath(ref, startPos, evaluator, ...);

// On subsequent ticks, continue computing
while (pathfinder.isComputing()) {
    // Process a limited number of nodes to avoid blocking the tick
    pathfinder.computePath(ref, motionController, probeData, 50, accessor);
    // yield or wait for next tick
}

// Once complete, retrieve the result
if (pathfinder.getProgress() == AStarBase.Progress.ACCOMPLISHED) {
    AStarNode path = pathfinder.getPath();
    // ... use the path
}

// CRITICAL: Clean up resources when done
pathfinder.clearPath();
```

### Anti-Patterns (Do NOT do this)
- **Blocking Computation:** Calling **computePath** in a loop within a single tick until it completes. This will freeze the server thread and cause severe lag if the path is complex.
- **State Reuse without Clearing:** Re-using an **AStarBase** instance by calling **initComputePath** without a corresponding **clearPath** call for the previous path can lead to subtle bugs, though **initComputePath** does perform a clear internally. However, forgetting to clear a failed or abandoned path will leak memory from the **AStarNodePool**.
- **External State Modification:** Directly adding or removing elements from the collections returned by **getOpenNodes** or **getVisitedBlocks** will corrupt the algorithm's state and cause undefined behavior.
- **Multi-threaded Access:** Invoking methods on a single **AStarBase** instance from multiple threads. This will cause state corruption and crashes.

## Data Pipeline
The flow of data for a single pathfinding operation is orchestrated by a higher-level AI system.

> Flow:
> AI Goal (e.g., MoveToTask) -> **AStarBase.initComputePath** -> (Looping) **AStarBase.computePath** -> Probes World via **MotionController** -> Generates/Updates **AStarNode** graph -> Path Found -> **AStarBase.getPath** -> Path consumed by Movement System -> Entity Motion Update

