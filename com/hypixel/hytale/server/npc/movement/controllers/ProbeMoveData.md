---
description: Architectural reference for ProbeMoveData
---

# ProbeMoveData

**Package:** com.hypixel.hytale.server.npc.movement.controllers
**Type:** Transient State Object

## Definition
```java
// Signature
public class ProbeMoveData {
```

## Architecture & Concepts

ProbeMoveData is a stateful data container that defines the parameters for, and captures the results of, a single NPC movement simulation. It acts as the primary data exchange object between high-level AI behaviors and the low-level MotionController responsible for path validation against world geometry.

This class embodies the **Parameter Object** pattern. Instead of a method signature like `probeMove(start, end, avoidDamage, savePath, ...)`, it encapsulates all configuration and state into a single, reusable structure. This decouples the AI's *intent* to move from the `MotionController`'s *algorithm* for simulating that movement.

The core architectural function of ProbeMoveData is to serve as a "worksheet" for a MotionController. The controller performs a series of physics steps and collision checks, recording each significant event (e.g., hitting a wall, starting a drop) as a Segment within the ProbeMoveData instance. The calling system can then inspect the final state of the object to determine if the path was successful and, if requested, reconstruct the exact path taken.

## Lifecycle & Ownership

-   **Creation:** A ProbeMoveData instance is created on-demand by a higher-level system that needs to validate a potential movement, such as an AI pathfinder or a specific scripted behavior. It is designed to be cheap to instantiate.

-   **Scope:** The object's lifetime is exceptionally short, typically confined to the scope of a single method call within an entity's update tick. It is created, configured, passed to a MotionController, evaluated, and then becomes eligible for garbage collection.

-   **Destruction:** The object is managed by the Java garbage collector. As it holds no persistent references and is not registered with any global systems, it is cleaned up as soon as it falls out of scope.

## Internal State & Concurrency

-   **State:** ProbeMoveData is highly **mutable**. Its primary purpose is to have its internal state modified by an external system (the MotionController) during the `probeMove` operation. The `probePosition` vector is updated continuously during the simulation, and the `segments` array is populated if enabled. This class is fundamentally a state machine for an in-flight query.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded access within the server's main entity processing loop. Concurrent calls to a `MotionController` using the same ProbeMoveData instance will result in race conditions and catastrophic state corruption.

## API Surface

The public API is divided into three categories: configuration, execution, and result inspection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setPosition(pos) | ProbeMoveData | O(1) | Sets the initial start point for the simulation. |
| setTargetPosition(target) | ProbeMoveData | O(1) | Sets the desired destination for the simulation. |
| setSaveSegments(bool) | void | O(1) | Configures whether to record a detailed path trace. |
| canMoveTo(...) | boolean | O(N) | Executes the probe via a MotionController. Complexity depends on the controller and path length. |
| canAdvance(...) | boolean | O(N) | Executes the probe and checks if a certain percentage of the path was completed. |
| computePosition(dist, result) | boolean | O(S) | Calculates a world position along the recorded path. S is the number of segments. |
| getLastDistance() | double | O(1) | Returns the total distance traveled by the successful portion of the probe. |

## Integration Patterns

### Standard Usage

The standard pattern involves creating a new ProbeMoveData instance for each movement query, configuring its start and end points, and passing it to the appropriate MotionController for execution. The result is then evaluated to inform AI decisions.

```java
// Assumes 'npc', 'controller', and 'accessor' are available in context
Vector3d start = npc.getPosition();
Vector3d destination = findNextWaypoint();

// 1. Create a transient data object for the query
ProbeMoveData probe = new ProbeMoveData();

// 2. Configure the probe's parameters
probe.setPosition(start);
probe.setTargetPosition(destination);
probe.setSaveSegments(false); // Do not need detailed path for simple validation

// 3. Execute the simulation via the MotionController
boolean isReachable = probe.canMoveTo(npc.getRef(), controller, MAX_PROBE_DISTANCE, accessor);

if (isReachable) {
    // The NPC can move towards the destination
    npc.getNavigator().setPath(probe.probePosition);
} else {
    // The path is blocked; trigger replanning
    npc.getAI().replanPath();
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use without Resetting:** Do not re-use a ProbeMoveData object for a new probe without explicitly re-configuring its start and target positions. The object retains its final state from the previous simulation, which will lead to incorrect calculations. Always treat it as a single-use object or be diligent about resetting it.

-   **Direct State Manipulation:** Do not manually call the `add...Segment` methods. These methods are part of the internal contract between this class and a MotionController implementation. Calling them directly will corrupt the probe's state machine.

-   **Asynchronous Execution:** Do not pass a ProbeMoveData instance to a MotionController on a background thread while the main thread might still hold a reference to it. The object is not designed for concurrent access.

## Data Pipeline

ProbeMoveData acts as a stateful courier in the NPC movement pipeline. It does not transform data itself but carries it between stages, accumulating results along the way.

> Flow:
> AI Behavior System (Defines Goal) -> **ProbeMoveData** (Initialized with Start/Target) -> MotionController.probeMove (Executes Simulation) -> World & Collision System (Provides Geometry Feedback) -> **ProbeMoveData** (Updated with Final Position & Path Segments) -> AI Behavior System (Reads Results to Make Decision)

