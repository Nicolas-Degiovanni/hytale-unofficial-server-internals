---
description: Architectural reference for SteeringForceWander
---

# SteeringForceWander

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient

## Definition
```java
// Signature
public class SteeringForceWander implements SteeringForce {
```

## Architecture & Concepts
SteeringForceWander is a concrete implementation of the SteeringForce strategy interface. It encapsulates the logic for generating a steering vector that simulates aimless, non-deliberate movement for server-side Non-Player Characters (NPCs).

This component does not directly manipulate an NPC's position or velocity. Instead, it acts as a "force generator" within the broader AI movement system. Its primary responsibility is to calculate a desired direction of travel. This output is then consumed by a higher-level steering behavior aggregator, which may combine it with other forces—such as collision avoidance or path following—before a final velocity is applied to the NPC's physics entity.

The internal logic uses a time-based jitter mechanism. Rather than generating a completely new random direction each frame, which would result in erratic movement, it maintains a persistent velocity vector and applies small, random perturbations over time. This produces a more natural, smoothed wandering effect.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level AI controller, such as a Behavior Tree or Finite State Machine, when an NPC is directed to enter a "wander" state. Each NPC actively wandering will have its own unique instance of SteeringForceWander.
- **Scope:** The object's lifetime is tightly coupled to the duration of the wander behavior. It persists as long as the NPC remains in that state.
- **Destruction:** The instance is eligible for garbage collection as soon as the NPC transitions to a different behavior (e.g., "seek target" or "idle"). It is not pooled or reused.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It internally maintains the current wander direction (velocity), a timer (time), and configuration parameters (turnInterval, jitter). This state is essential for producing coherent movement across multiple server ticks.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single thread that manages a specific NPC's AI updates. All method calls, particularly `updateTime` and `compute`, must be serialized. Concurrent access will lead to race conditions and unpredictable movement behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTurnTime(double t) | void | O(1) | Resets the internal timer and configures the interval for direction changes. |
| updateTime(double dt) | void | O(1) | Advances the internal simulation time. Must be called each update cycle. |
| setHeading(float heading) | void | O(1) | Overwrites the current velocity vector to face a specific direction. |
| compute(Steering output) | boolean | O(1) | Calculates the wander force for the current frame. Modifies the output parameter. |

## Integration Patterns

### Standard Usage
This component is intended to be used within an NPC's per-tick update loop. The controller is responsible for creating the instance, updating its timer, and calling compute.

```java
// Within an NPC's AI update method
// Assumes 'this.wanderForce' is an instance of SteeringForceWander

// 1. Update the internal timer with the delta time from the game loop
this.wanderForce.updateTime(deltaTime);

// 2. Prepare an output object to receive the steering data
Steering steeringOutput = new Steering();

// 3. Execute the wander logic
this.wanderForce.compute(steeringOutput);

// 4. Pass the result to the NPC's movement controller
npc.getMovementController().applySteering(steeringOutput);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a single SteeringForceWander instance for an NPC that has switched away from and then back to a wander behavior. The internal timer and velocity vector will be stale. Always create a new instance when the behavior is initiated.
- **Shared Instances:** Never share a single SteeringForceWander instance across multiple NPCs. Each NPC requires its own independent state to wander correctly.
- **Skipping Time Updates:** Failing to call `updateTime` on each tick will cause the wander logic to behave incorrectly, as the conditions for changing direction will never be met.

## Data Pipeline
The data flow for this component is unidirectional, transforming a time delta into a directional steering output.

> Flow:
> Server Game Loop Tick (Delta Time) -> **SteeringForceWander.updateTime()** -> **SteeringForceWander.compute()** -> Steering Output Object -> Movement Aggregator -> Final NPC Velocity

