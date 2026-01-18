---
description: Architectural reference for BodyMotionPath
---

# BodyMotionPath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class BodyMotionPath extends BodyMotionBase {
```

## Architecture & Concepts

The BodyMotionPath component is a high-level steering behavior responsible for guiding a Non-Player Character (NPC) along a predefined sequence of waypoints. It acts as the primary driver for patrol, guard, and other path-following AI behaviors.

Architecturally, this class functions as a stateful orchestrator. It does not compute low-level physics itself. Instead, it translates the abstract concept of a path—defined by an IPath asset—into concrete, frame-by-frame movement commands. It achieves this by managing and delegating to specialized **SteeringForce** components, primarily SteeringForcePursue for translation and SteeringForceRotate for orientation.

The core logic operates as a state machine, transitioning the NPC between several key states:
*   **Moving:** Actively pursuing the next point along the path.
*   **Waiting:** Paused at a waypoint for a configured duration.
*   **Observing:** While waiting, rotating to scan a defined view sector.
*   **Path Re-evaluation:** Selecting the next waypoint based on the configured pathing rules.

The behavior is heavily configured through its builder, allowing for significant variation in traversal logic via the **Shape** and **Direction** enumerations. These settings determine whether a path is treated as a linear route, a closed loop, a random set of points, or a chain that can link to other paths.

### Lifecycle & Ownership
- **Creation:** BodyMotionPath is instantiated by the server's asset loading pipeline via its corresponding builder, BuilderBodyMotionPath. It is never created directly. It is constructed as part of an NPC's Role definition, which encapsulates a set of behaviors.
- **Scope:** The component's lifetime is bound to the NPC entity and its active Role. It persists as long as the NPC is alive and this motion behavior is part of its current AI state. The internal state is reset via the activate and loaded methods when the component is initialized or re-initialized.
- **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded or destroyed. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This component is highly stateful and mutable. It maintains significant internal state to track its progress along a path, including the current waypoint index, target positions, delay timers, and boolean flags for internal state transitions like rotating or pendingNodeDelay. This state is essential for its frame-by-frame operation and is not intended to be modified externally.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single NPC within the main server game loop. All public methods, especially computeSteering, modify internal fields without synchronization.

    **WARNING:** Accessing a BodyMotionPath instance from multiple threads will lead to state corruption, race conditions, and unpredictable NPC behavior. All interactions must be synchronized with the owning entity's update tick.

## API Surface
The public contract is focused on lifecycle hooks and the primary steering computation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(ref, role, accessor) | void | O(1) | Lifecycle hook. Resets internal state when the component becomes active. |
| loaded(role) | void | O(1) | Lifecycle hook. Invalidates the current waypoint and resets state upon loading. |
| computeSteering(ref, role, info, dt, steering, accessor) | boolean | O(1) per frame | The primary update method. Calculates the desired steering for the current frame and populates the output Steering object. Returns false if no path is available or a new path is required. |

## Integration Patterns

### Standard Usage
BodyMotionPath is not used directly. It is configured within an NPC asset and managed by the NPC's Role system. The game engine's AI update loop invokes computeSteering on the active motion component for each NPC on every server tick.

```java
// Conceptual example of engine-level invocation
// This code would exist within an NPC's Role update logic.

// Assume 'activeMotion' is the current BodyMotionPath instance
// Assume 'steeringOutput' is a pre-allocated Steering object
boolean canContinue = activeMotion.computeSteering(
    entityRef,
    this, // The Role instance
    sensorInfoProvider,
    deltaTime,
    steeringOutput,
    componentAccessor
);

if (canContinue) {
    // Apply the computed steering to the NPC's motion controller
    motionController.applySteering(steeringOutput);
} else {
    // The component signals it cannot compute steering,
    // potentially triggering a new path request or behavior change.
    worldSupport.requestNewPath();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use a constructor to create a BodyMotionPath. The component is complex and must be configured and instantiated via its designated builder, BuilderBodyMotionPath, during asset loading.
- **External State Modification:** Do not attempt to read or write the internal state fields of this class (e.g., currentWaypointIndex, currentNodeDelay). This will break the internal state machine and cause erratic behavior. The component manages its own state exclusively through the computeSteering method.
- **Ignoring Return Value:** The boolean return value of computeSteering is critical. A false result indicates that the component has reached the end of a path or cannot proceed. Ignoring this signal can leave an NPC stranded and unable to request a new path.

## Data Pipeline
BodyMotionPath transforms static path data into continuous, dynamic steering commands.

> Flow:
> IPathProvider (Sensor Info) -> **BodyMotionPath**.computeSteering() -> Populated Steering Object -> MotionController -> NPC Transform Update

