---
description: Architectural reference for the Motion interface, the core contract for NPC movement behavior.
---

# Motion

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Behavioral Contract / Strategy Interface

## Definition
```java
// Signature
public interface Motion extends RoleStateChange, IAnnotatedComponent {
```

## Architecture & Concepts
The Motion interface is a fundamental contract within the server-side Non-Player Character (NPC) artificial intelligence framework. It embodies the **Strategy Pattern** to define discrete, interchangeable movement behaviors. Each implementation of Motion represents a specific way an NPC can move, such as wandering, fleeing, seeking a target, or standing still.

This interface acts as the critical bridge between high-level AI objectives, managed by a Role, and the low-level physics and movement systems. The core responsibility of a Motion implementation is to translate an abstract goal (e.g., "escape from the player") into a concrete, per-tick steering vector that can be applied to the NPC's physics component.

By decoupling the *intent* of a movement from its *execution*, the system allows for complex and dynamic AI. An NPC's active Motion can be swapped out at any time by its governing Role or state machine in response to sensory input or changing world conditions.

The interface inherits from RoleStateChange, signaling its deep integration with the NPC's state lifecycle, and IAnnotatedComponent, allowing for metadata-driven behavior and configuration.

### Lifecycle & Ownership
- **Creation:** Implementations of Motion are typically instantiated by the AI system, often within a factory or as part of a Role's state transition logic. They are not meant to be long-lived, shared objects. A new instance, for example a FleeMotion, might be created when an NPC's threat sensor is triggered.
- **Scope:** The lifetime of a Motion instance is strictly bound to the duration of the behavior it represents. It exists only as long as it is the *active* motion strategy for a given NPC.
- **Destruction:** An instance is eligible for garbage collection as soon as the NPC's AI controller transitions to a new state and replaces it with a different Motion implementation. Ownership is transient and managed entirely by the NPC's active Role.

## Internal State & Concurrency
- **State:** The Motion interface itself is stateless. Implementations are strongly encouraged to be stateless as well, receiving all necessary context via method parameters. This design promotes reusability and avoids complex state management. Any state that must be stored (e.g., a pathfinding result) should be considered temporary and scoped to the active duration of that motion.
- **Thread Safety:** **Not thread-safe.** The NPC AI update loop is presumed to be single-threaded on a per-world or per-region basis. All methods on a Motion instance are expected to be invoked sequentially from this single AI thread. Concurrent invocation of computeSteering for the same entity will lead to undefined behavior and race conditions on the output Steering object.

## API Surface
The public contract of Motion is centered around a state-based lifecycle and the core steering computation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| preComputeSteering(...) | void | Variable | Optional hook for expensive, ahead-of-time calculations like pathfinding. Called before the motion is fully active. |
| activate(...) | void | O(1) | Lifecycle callback invoked exactly once when this motion becomes the NPC's active strategy. Used for initialization. |
| deactivate(...) | void | O(1) | Lifecycle callback invoked exactly once when this motion is being replaced. Used for cleanup. |
| computeSteering(...) | boolean | Variable | **Core Method.** Called every AI tick to calculate the desired movement vector. Returns true if the motion is ongoing, false if it has completed its objective. |

## Integration Patterns

### Standard Usage
A high-level AI controller, such as a Role, manages the active Motion for an NPC. During the AI tick, it invokes computeSteering on the current Motion instance and applies the result to the entity's movement component.

```java
// Simplified example from within an NPC's AI update method
Motion currentMotion = npc.getActiveRole().getCurrentMotion();
Steering outputSteering = new Steering();

// The core interaction
boolean isStillActive = currentMotion.computeSteering(
    npc.getRef(),
    npc.getActiveRole(),
    npc.getInfoProvider(),
    deltaTime,
    outputSteering,
    npc.getComponentAccessor()
);

// Apply the calculated steering to the NPC
npc.getMovementController().applySteering(outputSteering);

if (!isStillActive) {
    // Transition to a new motion (e.g., IdleMotion)
    npc.getActiveRole().transitionToNextState();
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Avoid storing persistent world state within a Motion implementation. This violates the stateless design principle and can cause bugs when instances are reused or when state is not properly cleaned up on deactivation.
- **Ignoring the Lifecycle:** Do not call computeSteering on a Motion that has not been activated via the activate method. The lifecycle methods are not optional and guarantee the internal state is correct.
- **Ignoring the Return Value:** The boolean return from computeSteering is a critical signal. Ignoring it means the AI will never know when a motion (like "walk to point X") is complete, potentially leaving the NPC stuck in that state indefinitely.

## Data Pipeline
The Motion interface is a central processing stage in the NPC AI data flow, converting abstract goals into concrete actions.

> Flow:
> AI State Machine (Role) -> Selects **Motion** Implementation -> **Motion.computeSteering** (consumes World State) -> Produces Steering Object -> NPC Movement System -> Entity Transform Update

