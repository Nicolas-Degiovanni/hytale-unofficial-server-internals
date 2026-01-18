---
description: Architectural reference for AStarNode
---

# AStarNode

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Transient Data Model

## Definition
```java
// Signature
public class AStarNode implements IWaypoint {
```

## Architecture & Concepts

The AStarNode is the fundamental data structure for the server-side A* pathfinding algorithm. It represents a single, discrete point or "node" within the navigation search space, which is an implicit graph of traversable locations in the game world.

This class is more than a simple coordinate container. It encapsulates all the state required by the A* algorithm to evaluate paths, including:
*   **g-cost** (travelCost): The actual cost of the path from the starting node to the current node.
*   **h-cost** (estimateToGoal): The heuristic, or estimated, cost from the current node to the destination.
*   **f-cost** (totalCost): The sum of g-cost and h-cost. This value is the primary metric used by the pathfinder's priority queue to determine which node to explore next.

Nodes are linked together to form paths. The `predecessor` field creates a reverse-linked list, allowing the final path to be efficiently reconstructed by traversing backwards from the goal node once it is found. The `nextPathNode` field is populated after path reconstruction to create a forward-linked list, which is what an NPC entity consumes to follow the path.

The `IWaypoint` interface indicates that a finalized sequence of AStarNodes serves as a concrete navigation plan for higher-level AI and entity movement systems.

### Lifecycle & Ownership

-   **Creation:** AStarNode instances are created and managed exclusively by a pathfinding service, likely an AStarPathfinder. They are instantiated on-demand as the algorithm explores the world graph. The presence of multiple `init` methods strongly suggests that nodes are recycled from an object pool to mitigate garbage collection overhead during intensive pathfinding operations.
-   **Scope:** The lifetime of an AStarNode is strictly bound to a single pathfinding query. It exists only for the duration of the search algorithm's execution.
-   **Destruction:** Once a pathfinding query is complete, the nodes used for the search are considered stale. They are either reclaimed by the garbage collector or, more likely, returned to an object pool for re-initialization and reuse in a future query.

## Internal State & Concurrency

-   **State:** The AStarNode is a highly **mutable** object. Its core purpose is to hold and update the state of a search point. Fields such as `travelCost`, `totalCost`, and `predecessor` are frequently modified as the A* algorithm discovers more optimal routes to a given node.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is designed for high-performance use within a single-threaded pathfinding loop. Unsynchronized concurrent access will lead to state corruption and unpredictable, incorrect pathing results.

## API Surface

The public API is designed for control by a managing pathfinding system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initAsStartNode(...) | AStarNode | O(N) | Configures the node as the starting point of a new pathfinding search. Complexity is due to array initialization. |
| initWithPredecessor(...) | AStarNode | O(N) | Configures a newly discovered node, linking it to its parent. Complexity is due to array initialization. |
| initAsInvalid(...) | AStarNode | O(N) | Marks a node as untraversable or invalid, effectively removing it from consideration. |
| adjustOptimalPath(...) | void | O(S) | Recalculates costs when a shorter path to this node is found. Recursively propagates cost adjustments to its successors. S is the size of the successor subgraph. |
| close() | void | O(1) | Marks the node as fully evaluated and "closed" by the algorithm. |
| advance(skip) | AStarNode | O(N) | Traverses N steps forward along a finalized path. |
| next() | AStarNode | O(1) | Retrieves the next waypoint in a finalized path. |

## Integration Patterns

### Standard Usage

This class is never used in isolation. It is the core component managed by a higher-level pathfinding service. The service maintains an "open set" (typically a priority queue) of AStarNodes, continuously processing the one with the lowest `totalCost`.

```java
// Conceptual example within a pathfinder
// NOTE: This is a simplified representation. Do not use directly.

AStarNode startNode = nodePool.get().initAsStartNode(startPos, ...);
PriorityQueue<AStarNode> openSet = new PriorityQueue<>((a, b) -> Float.compare(a.getTotalCost(), b.getTotalCost()));
openSet.add(startNode);

while (!openSet.isEmpty()) {
    AStarNode current = openSet.poll();
    if (isGoal(current)) {
        // Path found, reconstruct it from predecessors
        return reconstructPath(current);
    }

    current.close();

    for (Neighbor neighbor : getNeighbors(current)) {
        // Create or update neighbor AStarNode instances
        // ...
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct State Manipulation:** Do not manually modify fields like `travelCost` or `predecessor` after initialization. The algorithm's correctness depends on the precise calculations performed by methods like `adjustOptimalPath`.
-   **Reusing Without Re-initialization:** Using an AStarNode instance from a previous pathfinding operation without calling one of the `init` methods will result in pathing failures due to stale state.
-   **Cross-Thread Access:** Never access or modify an AStarNode from a thread other than the one executing the pathfinding query. This will cause race conditions and corrupt the pathfinding state.

## Data Pipeline

AStarNode is a central processing component within the NPC navigation data pipeline. It does not source or sink data itself, but rather represents the state of the pathfinding process.

> Flow:
> Pathfinding Request -> World Graph Analysis -> **AStarNode** (Creation & Evaluation) -> Path Reconstruction -> Final Path (linked list of **AStarNode**s) -> NPC Movement System

