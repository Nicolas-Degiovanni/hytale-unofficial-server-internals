---
description: Architectural reference for BodyMotionMaintainDistance
---

# BodyMotionMaintainDistance

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionMaintainDistance extends BodyMotionBase {
```

## Architecture & Concepts
The BodyMotionMaintainDistance class implements a sophisticated "kiting" or "skirmishing" behavior for Non-Player Characters (NPCs). It is a specialized component within the server-side NPC AI framework, responsible for translating the high-level goal of maintaining a specific distance from a target into concrete steering outputs.

This component operates as a state machine with three primary logical states:
1.  **Approaching:** When the NPC is outside the maximum desired range, it will move towards the target using a SteeringForcePursue behavior.
2.  **Retreating:** When the NPC is inside the minimum desired range, it will move away from the target using a SteeringForceEvade behavior.
3.  **Maintaining:** When within the desired range, the NPC ceases aggressive pursuit or evasion. In this state, it can engage in secondary movements like strafing to remain mobile and evasive.

It acts as a crucial link between the NPC's sensory system (InfoProvider) and its physical actuation system (MotionController). It consumes target position data, processes it against its configured distance parameters, and produces a desired steering vector and yaw. This output is then passed to a MotionController, such as MotionControllerWalk, which handles the final physics-aware execution of the movement.

### Lifecycle & Ownership
-   **Creation:** An instance of BodyMotionMaintainDistance is created by the NPC asset loading system via its corresponding builder, BuilderBodyMotionMaintainDistance. It is **not** intended for direct instantiation. The builder injects all configuration parameters, such as distance ranges and movement speeds, which are defined in NPC asset files.
-   **Scope:** The lifecycle of this object is tied to an NPC's active AI state. It persists only as long as the NPC is required to perform the "maintain distance" behavior. When the NPC's AI state machine transitions to a different behavior (e.g., patrolling, fleeing unconditionally), this instance is discarded.
-   **Destruction:** The deactivate method is invoked by the NPC's Role manager when this component is no longer the active motion strategy. This ensures a clean state transition, for example, by resetting the backingAway flag on the NPC's Role. The Java garbage collector is responsible for final memory reclamation.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains internal flags like *approaching* and *movingAway* to track its current tactical mode. It also manages timers and state for strafing behavior (strafingDelay, strafingDirection) and caches references to parameter providers from the sensor system for performance. All state is specific to a single NPC's interaction with a single target.

-   **Thread Safety:** This component is **not thread-safe** and must only be accessed from the server's main entity update thread. Its design assumes a single-threaded, sequential execution model where computeSteering is called once per NPC per game tick. Concurrent access would lead to severe race conditions and unpredictable NPC movement.

## API Surface
The public contract is minimal, centered on the per-tick update method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | The core update method. Calculates and applies steering forces to maintain distance. Populates the output Steering object. |
| deactivate(...) | void | O(1) | Cleans up the component's state when it is no longer active, resetting flags on the parent NPC Role. |

## Integration Patterns

### Standard Usage
This component is managed by an NPC's Role or a higher-level AI controller. During the NPC's update tick, the controller invokes computeSteering on the currently active BodyMotion component.

```java
// Within an NPC's AI update logic
// 'currentMotion' is an instance of BodyMotionMaintainDistance
Steering desiredSteering = new Steering();
currentMotion.computeSteering(
    entityRef,
    npcRole,
    sensorInfo,
    deltaTime,
    desiredSteering,
    componentAccessor
);

// The populated 'desiredSteering' is then passed to the motion controller
npcRole.getActiveMotionController().setDesiredSteering(desiredSteering);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BodyMotionMaintainDistance()`. The component relies on its corresponding Builder for proper initialization with asset-defined parameters. Bypassing the builder will result in a misconfigured or non-functional component.
-   **State Reuse:** Do not reuse an instance of this class for a different NPC or a different target without a full deactivation and re-initialization cycle. The internal state is highly specific to its context.
-   **Concurrent Modification:** Never call computeSteering from any thread other than the main server tick thread for the entity in question.

## Data Pipeline
This component functions as a processor that transforms sensory input into motion commands.

> Flow:
> InfoProvider (Target Position) -> **BodyMotionMaintainDistance.computeSteering** -> Steering Object (Vector, Yaw) -> MotionController -> Physics Engine -> Final Entity Transform Update

