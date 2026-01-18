---
description: Architectural reference for BodyMotionLeave
---

# BodyMotionLeave

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionLeave extends BodyMotionFindBase<AStarBase> {
```

## Architecture & Concepts
BodyMotionLeave is a server-side movement strategy component that directs a Non-Player Character (NPC) to move away from its starting position until a specified distance has been covered. It functions as a concrete implementation of a "flee" or "disperse" behavior within the broader NPC motion and AI framework.

Unlike typical pathfinding goals which seek the *shortest* path to a destination, BodyMotionLeave re-purposes the A* pathfinding algorithm to find the *furthest* possible path from an origin point. It does not have a target destination; its only goal is to achieve separation.

This component acts as a goal evaluator and path generation strategy for its superclass, BodyMotionFindBase. The core of its logic is encapsulated in two functions:
1.  A pathfinding directive that instructs the AStarBase algorithm to build a path of maximum length.
2.  A termination condition, `isGoalReached`, that continuously checks if the NPC's current position is far enough from its starting point.

The required distance is configured via an asset builder, allowing designers to specify flee distances on a per-NPC or per-behavior basis.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's NPC asset loading and behavior system. The constructor requires a `BuilderBodyMotionLeave` object, indicating it is created dynamically based on NPC configuration files, not hard-coded. This typically occurs when an NPC's AI State Machine or Behavior Tree transitions into a state requiring this movement pattern.
-   **Scope:** This object is short-lived and behavior-scoped. It exists only for the duration of a single "leave" action. Once the NPC has reached the target distance or the behavior is interrupted, this object is no longer referenced.
-   **Destruction:** The object is eligible for garbage collection as soon as the owning motion controller or AI behavior completes and discards its reference. It is not designed to be pooled or reused.

## Internal State & Concurrency
-   **State:** The primary internal state is the `distanceSquared` field. This state is **immutable**, as it is a final field set only once in the constructor. This design ensures that a given instance of BodyMotionLeave always represents the exact same movement goal it was created for.
-   **Thread Safety:** This class is **not thread-safe** and must only be used within the context of its owning NPC's update tick. The methods operate on mutable, externally-owned objects like MotionController and AStarBase. Concurrent access would lead to race conditions and severe corruption of pathfinding data. The server's entity update loop provides the necessary thread containment.

## API Surface
The public API provides hooks for a motion execution system to manage the pathfinding and goal-checking process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isGoalReached(...) | boolean | O(1) | Determines if the NPC is at or beyond the required distance from its start point. This is the termination condition for the movement. |
| estimateToGoal(...) | float | O(1) | Returns a pathfinding heuristic. Always returns 0.0, as the goal is not a specific point but rather maximum distance. |
| findBestPath(...) | void | O(N log N) | Instructs the provided AStarBase instance to generate the furthest possible path. Complexity is dependent on the A* implementation. |

## Integration Patterns

### Standard Usage
BodyMotionLeave is intended to be used by higher-level AI systems. A behavior node or state will construct it from asset data and hand it off to a generic motion controller, which then drives the execution of the movement.

```java
// Conceptual example within an AI behavior update
// Note: Actual instantiation is handled by the asset building system.

// 1. The AI system creates the BodyMotionLeave component from a definition.
BodyMotionLeave leaveBehavior = behaviorFactory.create(npcDefinition.getLeaveBehavior());

// 2. The component is passed to the NPC's motion controller.
MotionController controller = npc.getMotionController();
controller.setMovementGoal(leaveBehavior);

// 3. In subsequent server ticks, the controller manages the process.
// The controller will call findBestPath once, then isGoalReached every tick.
while (!controller.isGoalReached()) {
    controller.updateMovement();
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BodyMotionLeave()`. The class relies on the `BuilderBodyMotionLeave` and `BuilderSupport` context provided by the engine's asset pipeline to be configured correctly. Manual creation will result in an unconfigured or misconfigured component.
-   **State Reuse:** Do not attempt to reuse a BodyMotionLeave instance for a new movement task. The underlying `AStarBase` object it manages contains state from the previous pathfinding calculation and must be re-initialized. Always create a new instance for each new "leave" action.
-   **External Path Generation:** Do not call `findBestPath` and then attempt to use a different method for goal checking. The path generation and the `isGoalReached` logic are tightly coupled around the concept of distance from the *original* start position.

## Data Pipeline
BodyMotionLeave acts as a strategic controller in the NPC movement data flow. It does not transform data itself, but rather directs other systems based on its configuration.

> Flow:
> NPC AI State Change -> Asset Definition (`BuilderBodyMotionLeave`) -> **BodyMotionLeave Instantiation** -> MotionController -> `findBestPath` call -> AStarBase Path Calculation -> NPC Position Update -> `isGoalReached` check -> AI State Change (Success/Failure)

