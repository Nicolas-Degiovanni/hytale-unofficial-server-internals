---
description: Architectural reference for HeadMotionObserve
---

# HeadMotionObserve

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Component (Stateful Behavior)

## Definition
```java
// Signature
public class HeadMotionObserve extends HeadMotionBase {
```

## Architecture & Concepts
The HeadMotionObserve class is a server-side component that implements a specific NPC head movement behavior: observing or scanning an area. It functions as a state machine to create lifelike, idle head-turning motions, such as systematically sweeping a view cone or making random glances.

As a concrete implementation of HeadMotionBase, this component is designed to be managed by a higher-level NPC AI system, such as a Role or a Behavior Tree. Its sole responsibility is to compute the desired head yaw for a single frame based on its internal state and configuration. It does not influence the NPC's body position or orientation; it only generates rotational steering data for the head.

This component is highly configurable through its corresponding builder, BuilderHeadMotionObserve. This allows designers to specify parameters like the total angle of the sweep, the pause duration between movements, and whether the head should move to random points or sweep through predefined segments. Internally, it composes a SteeringForceRotate object to translate its target angle into a smooth rotational force that can be consumed by the NPC's physics and animation systems.

## Lifecycle & Ownership
- **Creation:** An instance of HeadMotionObserve is not created directly. It is instantiated by the server's NPC asset loading pipeline, using a BuilderHeadMotionObserve. This typically occurs when an NPC's behavioral profile (its Role) is loaded into memory.
- **Scope:** The object's lifetime is tied to the NPC's active behavior. It persists as long as the NPC is in a state that requires this "observe" motion. When the AI transitions to a different state (e.g., combat, pathfinding), this component is deactivated. The `activate` method is called each time the behavior becomes active, resetting its internal state machine.
- **Destruction:** The object is eligible for garbage collection when the parent Role or behavior state is unloaded or replaced. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This component is stateful and highly mutable. While its core configuration parameters (angleRange, pauseTimeRange) are final and set at construction, its operational state is constantly changing. It maintains internal timers like preDelay and delay, a segment counter, and the current targetBodyOffsetYaw.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single NPC update thread. All internal state is unsynchronized. Any concurrent access to its methods, particularly computeSteering, will result in race conditions, corrupted state, and undefined behavior.

**WARNING:** Do not share instances of this class across multiple NPCs or access them from any thread other than the owning entity's update thread.

## API Surface
The public API is minimal, reflecting its role as an internal component driven by the AI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(ref, role, accessor) | void | O(1) | Resets and initializes the internal state machine. Must be called before the first call to computeSteering. |
| computeSteering(ref, role, info, dt, steering, accessor) | boolean | O(1) | The primary update method. Calculates the desired head rotation for the current frame and applies it to the output Steering parameter. |

## Integration Patterns

### Standard Usage
HeadMotionObserve is intended to be used as part of an NPC's behavior state. The controlling system (e.g., a Role) calls `activate` upon entering the "observe" state and then calls `computeSteering` on every subsequent server tick.

```java
// Within an NPC's AI update loop

// When the "Observe" state is first activated:
HeadMotionObserve headMotion = npc.getActiveHeadMotion();
headMotion.activate(entityRef, currentRole, componentAccessor);

// On each subsequent tick while in the "Observe" state:
Steering desiredSteering = new Steering();
headMotion.computeSteering(entityRef, currentRole, sensorInfo, deltaTime, desiredSteering, componentAccessor);

// The AI system then consumes 'desiredSteering' to update the entity's components.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new HeadMotionObserve()`. The component is complex and must be configured via its builder during asset loading. Direct instantiation will result in a non-functional or improperly configured object.
- **State Tampering:** Do not attempt to modify the internal state of the component from outside the class. The timers and target angles are managed by an internal state machine, and external interference will break the behavior.
- **Ignoring The `activate` Call:** Failing to call `activate` when the behavior becomes active will result in the component operating on stale or uninitialized state, leading to unpredictable movement.

## Data Pipeline
The component functions as a processor in the NPC's per-tick AI data flow. It reads entity state, processes it through its internal logic, and outputs a steering directive.

> Flow:
> 1.  **Input:** Reads current entity orientation from `TransformComponent` and `HeadRotation`.
> 2.  **Input:** Reads entity model constraints from `ModelComponent`.
> 3.  **Process:** The internal state machine in **HeadMotionObserve** determines the `targetBodyOffsetYaw` based on timers and configuration (random vs. segmented).
> 4.  **Process:** The internal `SteeringForceRotate` calculates the necessary rotational force to move the head towards the target yaw.
> 5.  **Output:** Modifies the `Steering` object passed into `computeSteering`, setting the desired yaw or turn speed.
> 6.  **Downstream:** The NPC's core AI system applies the final `Steering` data to the entity's `HeadRotation` component, which is then synchronized to clients.

