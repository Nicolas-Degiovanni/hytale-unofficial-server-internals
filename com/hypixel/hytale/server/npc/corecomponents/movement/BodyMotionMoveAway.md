---
description: Architectural reference for BodyMotionMoveAway
---

# BodyMotionMoveAway

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient

## Definition
```java
// Signature
public class BodyMotionMoveAway extends BodyMotionFindWithTarget {
```

## Architecture & Concepts
BodyMotionMoveAway is a high-level steering behavior component that implements a "flee" or "evade" strategy for Non-Player Characters (NPCs). It operates within the server-side NPC AI framework as a specific type of **BodyMotion**, a system responsible for translating abstract goals into concrete steering forces.

This component's primary function is to direct an NPC to move away from a designated target position. It builds upon the base functionality of BodyMotionFindWithTarget, inheriting the mechanism for tracking a target, but inverts the objective from seeking to evading.

To create more natural and less predictable movement, BodyMotionMoveAway employs several key strategies:

*   **Directional Persistence:** Instead of constantly recalculating a path directly away from the target, the component selects a general "flee direction" and maintains it for a randomized duration. This prevents the robotic, perfectly opposite movement often seen in simpler AI.
*   **Erratic Jitter:** When the NPC is within a configurable "erratic distance" to the target, the movement becomes more panicked and unpredictable. The component applies a larger random jitter to its chosen flee direction and shortens the duration it holds that direction, simulating a creature's desperate attempt to escape close-range danger.
*   **View Sector Trigger:** The component can be configured to re-evaluate its flee direction immediately if the target enters the NPC's forward-facing view sector, simulating the NPC reacting to visually reacquiring the threat.

Internally, it delegates the final steering vector calculation to a specialized **SteeringForceEvade** object, which handles the low-level physics and vector mathematics. BodyMotionMoveAway acts as the policy layer, deciding *when* and *how* to flee, while SteeringForceEvade determines the precise force required.

## Lifecycle & Ownership
- **Creation:** Instances of BodyMotionMoveAway are not created directly via code. They are instantiated by the NPC asset loading pipeline, configured through a corresponding **BuilderBodyMotionMoveAway** definition in an NPC's behavior assets. This ensures that behaviors are data-driven and decoupled from game logic.

- **Scope:** The lifecycle of a BodyMotionMoveAway instance is tied to the NPC's active behavioral state. It is activated when an NPC's state machine or role manager transitions into a state that requires fleeing (e.g., "Fear", "Retreating"). It persists only as long as that state is active.

- **Destruction:** The object is eligible for garbage collection once the NPC's active **Role** releases its reference, typically upon transitioning to a different behavior (e.g., "Idle", "Attacking"). There is no manual destruction method; its lifetime is managed entirely by the AI state machine.

## Internal State & Concurrency
- **State:** This component is highly stateful and mutable. It maintains internal state variables such as **holdDirectionTimeRemaining**, the current **fleeDirection**, and configuration parameters like **stopDistance** and **jitterAngle**. This state is modified on nearly every call to computeSteering.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single NPC within the server's main AI update loop. All public methods, particularly computeSteering, modify internal state without any synchronization mechanisms.

    **Warning:** Concurrent access from multiple threads will lead to state corruption, race conditions, and unpredictable NPC behavior. Under no circumstances should an instance of this class be shared between NPCs or accessed from outside its designated AI update thread.

## API Surface
The public contract is designed for interaction with the NPC's Role and MotionController systems, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(ref, role, accessor) | void | O(1) | Resets the component's internal state. Called by the AI system when this behavior becomes active. |
| computeSteering(ref, role, info, dt, steering, accessor) | boolean | O(1) | The core update method. Calculates the desired steering force for the current tick and populates the output Steering parameter. |
| isGoalReached(...) | boolean | O(1) | Determines if the component's objective is met. Returns true if the NPC is beyond the configured stopDistance from the target. |
| estimateToGoal(aStarBase, from, controller) | float | O(1) | Provides a cost heuristic for the A* pathfinding system, representing the remaining distance to the "safe" zone. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly. Its behavior is triggered by the NPC's state machine. A developer would configure its parameters in an asset file and then assign a state to an NPC that utilizes this BodyMotion. The system handles the rest.

The following example is a conceptual representation of how the *engine* uses the component, not how a developer would.

```java
// Conceptual engine-level usage within an NPC's update tick
// This code would exist inside a Role or AI state manager.

// Assuming 'currentMotion' is an active BodyMotionMoveAway instance
Steering desiredSteering = new Steering();
boolean hasSteering = currentMotion.computeSteering(npcRef, this, info, dt, desiredSteering, accessor);

if (hasSteering) {
    npcMotionController.applySteering(desiredSteering);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BodyMotionMoveAway()`. This bypasses the data-driven configuration from the asset builder and will result in an uninitialized or incorrectly configured component.
- **State Sharing:** Do not share a single instance of BodyMotionMoveAway across multiple NPCs. Its internal state is specific to one NPC's context (e.g., its current flee direction and timer).
- **External State Modification:** Do not modify public fields or internal state variables from outside the class. Rely solely on the computeSteering method to drive state changes.

## Data Pipeline
The flow of data through this component during a single game tick is linear and synchronous. It transforms high-level target information into a low-level steering command.

> Flow:
> Target Position (from InfoProvider) -> **BodyMotionMoveAway.computeSteering** -> Steering Vector (output parameter) -> MotionController -> Physics System -> Updated NPC Transform

