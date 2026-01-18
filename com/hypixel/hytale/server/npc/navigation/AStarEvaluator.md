---
description: Architectural reference for AStarEvaluator
---

# AStarEvaluator

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface AStarEvaluator {
```

## Architecture & Concepts
The AStarEvaluator interface defines a behavioral contract for the core logic of the A* pathfinding algorithm. It embodies the Strategy Pattern, decoupling the generic A* search mechanism, likely implemented in a class like AStarBase, from the specific goal-definition and cost-estimation logic of a given pathfinding query.

This interface is fundamental to all NPC navigation. It answers two critical questions for the pathfinder:
1.  **Termination Condition:** "Have we arrived at the destination?" (isGoalReached)
2.  **Heuristic Function:** "What is the estimated cost to reach the destination from the current location?" (estimateToGoal)

By abstracting this logic, the engine can use the same core A* search algorithm to navigate entities to a specific point, to a type of block, or towards another entity, simply by providing a different AStarEvaluator implementation.

## Lifecycle & Ownership
As an interface, AStarEvaluator itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

-   **Creation:** Implementations are instantiated by systems that require pathfinding services. They are typically created on-demand for a specific navigation task and passed as a parameter to the constructor or initialization method of the primary pathfinding class (e.g., AStarBase).
-   **Scope:** The lifetime of an AStarEvaluator implementation is tightly coupled to the pathfinding job it services. It persists only for the duration of a single path calculation.
-   **Destruction:** Once the pathfinding algorithm completes or is terminated, the AStarEvaluator instance is no longer referenced by the pathfinder and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The interface contract implies that implementations should be stateless. Any state held within an implementation should be considered immutable for the duration of a single pathfinding operation. Caching results within an evaluator is a potential optimization but introduces significant complexity and risk.
-   **Thread Safety:** **WARNING:** Pathfinding is a computationally expensive task and is frequently offloaded to background worker threads. Implementations of AStarEvaluator **must be thread-safe** or, preferably, designed to be instantiated per-job to avoid shared state altogether. A shared, stateful evaluator is a primary source of concurrency bugs, such as race conditions and inconsistent pathing results.

## API Surface
The public contract consists of the two methods required to guide the A* search algorithm.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isGoalReached(...) | boolean | O(1) | Determines if the provided AStarNode satisfies the goal condition. This is the termination check for the pathfinder. |
| estimateToGoal(...) | float | O(1) | Provides the heuristic estimate (h) of the cost from a given point to the final goal. Must be an admissible heuristic. |

## Integration Patterns

### Standard Usage
An implementation of AStarEvaluator is never called directly by application code. Instead, it is supplied to a pathfinding controller, which then uses it internally during its search loop.

```java
// Example: Creating a pathfinder with a specific evaluation strategy
// NOTE: PointEvaluator is a hypothetical implementation of AStarEvaluator
AStarEvaluator pointEvaluator = new PointEvaluator(targetPosition);
AStarBase pathfinder = new AStarBase(world, startNode, pointEvaluator);

// The pathfinder now internally calls pointEvaluator.isGoalReached() and
// pointEvaluator.estimateToGoal() during its execution.
Path result = pathfinder.findPath();
```

### Anti-Patterns (Do NOT do this)
-   **Non-Admissible Heuristics:** The `estimateToGoal` method must never overestimate the actual cost to the goal. Doing so violates the admissibility requirement of A*, breaking its guarantee of finding the optimal path and potentially leading to highly inefficient or incorrect routes.
-   **Expensive Computations:** Both methods are called frequently, often thousands of times for a single path request. Performing file I/O, network requests, or complex world queries within these methods will severely degrade server performance.
-   **Stateful Implementations:** Avoid storing mutable state within an evaluator that is shared across multiple pathfinding jobs. This will lead to unpredictable behavior and severe concurrency issues.

## Data Pipeline
AStarEvaluator does not process a stream of data but rather acts as a queryable logic component within the A* algorithm's control flow.

> Flow:
> Pathfinding Request -> AStarBase Search Loop -> **AStarEvaluator.estimateToGoal()** -> Priority Queue Update -> AStarBase Search Loop -> **AStarEvaluator.isGoalReached()** -> Path Result or Continued Search

