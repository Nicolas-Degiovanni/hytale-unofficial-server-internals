---
description: Architectural reference for SteeringForceEvade
---

# SteeringForceEvade

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient

## Definition
```java
// Signature
public class SteeringForceEvade extends SteeringForceWithTarget {
```

## Architecture & Concepts
The SteeringForceEvade class is a fundamental component within the server-side NPC autonomous movement system. It implements a sophisticated "evade" or "flee" behavior based on classic steering force principles. This is not a simple repulsion; it defines a behavior that varies with distance, creating more natural and less robotic movement.

This class operates as a single, isolated force calculation module. In the broader AI architecture, a high-level behavior controller (e.g., a state machine or behavior tree) instantiates and configures this class when an NPC needs to flee from a target. The resulting force vector calculated by this class is typically combined with other forces—such as obstacle avoidance or path following—in a weighted sum to produce the final acceleration for the NPC in a given game tick.

The core of its logic revolves around two key radii from the target:
1.  **Slowdown Distance:** The inner radius. Inside this zone, the NPC will attempt to flee at maximum force.
2.  **Stop Distance:** The outer radius. As the NPC moves from the slowdown distance to the stop distance, the magnitude of the evasion force is gradually reduced, or "damped". Once the NPC is beyond the stop distance, this force returns a zero vector, effectively deactivating the behavior.

This creates a smooth falloff effect, preventing abrupt stops once the NPC has successfully escaped. The class also supports a `directionHint`, allowing AI directors to suggest an evasion path, which is critical for guiding NPCs around environmental hazards while fleeing.

## Lifecycle & Ownership
-   **Creation:** An instance of SteeringForceEvade is created on-demand by a higher-level AI behavior system, such as an `EvadeBehavior` state or a node in a behavior tree. It is not a globally shared service.
-   **Scope:** The object's lifetime is tightly coupled to the duration of the specific evasion behavior. It persists only as long as the NPC is actively evading a target. When the NPC's state changes (e.g., to "idle" or "attack"), the instance is discarded.
-   **Destruction:** As a plain Java object with no external resource handles, it is managed entirely by the garbage collector. It is eligible for collection as soon as the owning AI behavior is deactivated and no longer holds a reference to it.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. Its behavior is defined by internal fields like `slowdownDistance`, `stopDistance`, and `directionHint`, all of which can be modified after instantiation. It also relies on mutable state inherited from its parent, `SteeringForceWithTarget`, which holds the positions of the NPC and its target.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used exclusively within the context of a single NPC's update cycle, which is expected to run on the main server thread. Concurrent calls to `compute` or its setters will lead to race conditions and unpredictable behavior.

## API Surface
The public API is designed for configuration and execution of the force calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compute(Steering output) | boolean | O(1) | The primary execution method. Calculates the evasion vector based on internal state and writes the result to the `output` parameter. Returns false if no force should be applied. |
| setDistances(min, max) | void | O(1) | Configures the slowdown and stop radii, which are the primary determinants of the force's behavior. This method also pre-calculates squared distances for performance. |
| setDirectionHint(heading) | void | O(1) | Provides a suggested directional heading for evasion, overriding the default behavior of moving directly away from the target. |
| setAdhereToDirectionHint(boolean) | void | O(1) | If true, forces the evasion vector to align strictly with the `directionHint`. |

## Integration Patterns

### Standard Usage
The intended use is to create, configure, and execute this force as part of a larger AI behavior update loop. The `output` parameter is a mutable `Steering` object, a common performance pattern to avoid object allocation on each tick.

```java
// Within an AI Behavior's update method...

// 1. Instantiate or retrieve the force object
SteeringForceEvade evadeForce = new SteeringForceEvade();

// 2. Configure its parameters
evadeForce.setTarget(enemy.getPosition());
evadeForce.setSelf(thisNpc.getPosition());
evadeForce.setDistances(10.0, 15.0); // Slow down inside 10m, stop beyond 15m

// 3. Create an output object and compute the force
Steering desiredSteering = new Steering();
boolean forceApplied = evadeForce.compute(desiredSteering);

if (forceApplied) {
    // 4. Add the resulting force to an accumulator
    npcMovementController.addForce(desiredSteering.getTranslation());
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not reuse a single `SteeringForceEvade` instance for multiple NPCs without re-configuring its target and self-position for each one. The internal state is specific to one NPC's context.
-   **Stale Positional Data:** Failing to update the `targetPosition` and `selfPosition` (via methods inherited from `SteeringForceWithTarget`) before each call to `compute` will result in calculations based on outdated information.
-   **Incorrect Distance Configuration:** Setting `slowdownDistance` to be greater than or equal to `stopDistance` will break the falloff logic and lead to undefined behavior.

## Data Pipeline
SteeringForceEvade acts as a pure processing node in the NPC movement data flow. It transforms positional data into a directional force vector.

> Flow:
> AI Behavior Controller (provides NPC & Target Positions) -> **SteeringForceEvade.compute()** -> Steering Vector (output) -> Force Accumulator -> Final NPC Velocity Calculation

