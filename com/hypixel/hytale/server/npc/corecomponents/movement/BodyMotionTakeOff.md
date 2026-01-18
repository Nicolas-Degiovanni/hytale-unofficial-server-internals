---
description: Architectural reference for BodyMotionTakeOff
---

# BodyMotionTakeOff

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Component

## Definition
```java
// Signature
public class BodyMotionTakeOff extends BodyMotionBase {
```

## Architecture & Concepts
BodyMotionTakeOff is a specialized, single-purpose component within the Non-Player Character (NPC) Artificial Intelligence framework. It represents a discrete, transitional action: initiating flight. It does not calculate ongoing movement or steering; instead, its sole responsibility is to orchestrate the state change from a grounded or non-flying motion controller to the MotionControllerFly.

This class is a concrete implementation of the BodyMotionBase, placing it within a strategy pattern where different BodyMotion objects define specific physical behaviors (e.g., walking, jumping, strafing, taking off). It acts as a command or an initiator, bridging the high-level behavioral logic (e.g., a Behavior Tree deciding to "flee by flying") and the low-level physics simulation managed by a MotionController.

The `computeSteering` method's `return false;` is a critical design choice. It signals to the calling system that this component does not produce a steering vector. Its work is completed within a single invocation by setting up the new state, after which a different BodyMotion component should take over for subsequent updates.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively via its corresponding builder, BuilderBodyMotionTakeOff. This builder is typically invoked by a higher-level AI definition system when constructing an NPC's behavior graph or state machine. The resulting BodyMotionTakeOff instance is owned by the specific AI state or node that uses it.
-   **Scope:** Extremely short-lived. This object is intended to be active for only a single tick or until the state transition to flying is confirmed. Once the MotionControllerFly is active, this component has fulfilled its purpose and should be discarded.
-   **Destruction:** The object is eligible for garbage collection as soon as the NPC's AI state machine transitions away from the "take-off" behavior and the reference to it is dropped.

## Internal State & Concurrency
-   **State:** The component is effectively immutable after construction. It holds a single configuration value, `jumpSpeed`, which is a final field initialized in the constructor. It does not cache data or maintain state across multiple calls. All state modifications are performed on external components like Role and TransformComponent.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be executed exclusively within the single-threaded server update loop. The `computeSteering` method directly accesses and mutates shared components via a ComponentAccessor, an operation that is fundamentally unsafe if performed concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | Checks the NPC's active motion controller. If not MotionControllerFly, it performs a state transition to activate it and triggers its `takeOff` method. Always returns false. |

## Integration Patterns

### Standard Usage
This component is used as a state or action within a larger AI system, such as a Behavior Tree or a Finite State Machine. It is activated when the AI decides the NPC should begin flying.

```java
// Conceptual usage within an AI system's update tick
// Assume 'currentMotion' is an active BodyMotionTakeOff instance

boolean providesSteering = currentMotion.computeSteering(
    entityRef,
    npcRole,
    sensorInfo,
    deltaTime,
    desiredSteering,
    accessor
);

// Because it returns false, the AI system knows the state has changed
// and should transition to a new BodyMotion (e.g., BodyMotionFlyToTarget)
// on the next tick.
if (!providesSteering) {
    npcRole.setActiveBodyMotion(new BodyMotionFlyToTarget(...));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BodyMotionTakeOff()`. The class requires its associated builder, BuilderBodyMotionTakeOff, to ensure all dependencies like `jumpSpeed` are correctly provided and validated.
-   **Persistent Activation:** Do not keep an instance of BodyMotionTakeOff as the active motion component for more than one tick. Its logic is not idempotent and will repeatedly attempt to switch motion controllers, causing performance overhead and undefined behavior. It is a transitional tool, not a continuous controller.

## Data Pipeline
BodyMotionTakeOff functions as a control-flow component rather than a data-processing one. It receives a signal to execute and, in turn, issues commands to other systems.

> Flow:
> AI Behavior System (activates state) -> **BodyMotionTakeOff.computeSteering()** -> Role.setActiveMotionController("Fly") -> MotionControllerFly.takeOff() -> TransformComponent (velocity updated)

