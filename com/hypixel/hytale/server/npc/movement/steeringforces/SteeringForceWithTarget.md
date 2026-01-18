---
description: Architectural reference for SteeringForceWithTarget
---

# SteeringForceWithTarget

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient

## Definition
```java
// Signature
public abstract class SteeringForceWithTarget implements SteeringForce {
```

## Architecture & Concepts
SteeringForceWithTarget is an abstract base class that serves as the foundation for all steering behaviors driven by a target position. It operates within the server-side NPC movement and AI system.

This class specializes the generic SteeringForce contract by introducing the core concept of a *self* and a *target*. It encapsulates the common state and pre-processing logic required by concrete implementations, such as a force to seek a target or flee from one.

Architecturally, it employs the **Template Method** design pattern. The `compute` method in this base class performs a critical pre-processing step—applying a component selector mask—before the logic of any concrete subclass is executed. This ensures that all target-based forces can be constrained to specific axes (e.g., forcing ground-based NPCs to ignore the Y-axis) in a consistent manner.

## Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses (e.g., SeekForce) are created by higher-level AI controllers, typically on a per-NPC, per-behavior basis. These instances are often short-lived.
- **Scope:** The lifetime of a SteeringForceWithTarget instance is typically bound to a single steering calculation within a single game tick. It is a stateful but ephemeral object used to compute a single movement vector. Systems may pool these objects for performance, but their logical scope remains one-shot.
- **Destruction:** Instances are eligible for garbage collection immediately after their contribution to the final steering vector has been calculated, unless managed by an object pool.

## Internal State & Concurrency
- **State:** Highly mutable. The class's primary purpose is to hold the transient state of an NPC's position and its target's position for the duration of a single calculation. The internal Vector3d objects are modified directly via setter methods.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be confined to a single thread, such as the server's main entity update loop. Concurrent modification of the self or target positions from multiple threads will lead to race conditions and unpredictable behavior. All access must be externally synchronized by the calling system.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setPositions(self, target) | void | O(1) | Sets the world-space positions for the entity and its target. This is the primary method for configuring the force. |
| setComponentSelector(selector) | void | O(1) | **CRITICAL:** Applies an axial mask to the calculation. A null selector will cause a NullPointerException. |
| compute(output) | boolean | O(1) | Pre-processes internal state by applying the component selector. Subclasses must override this to implement the actual force calculation. |

## Integration Patterns

### Standard Usage
A concrete implementation should be instantiated, configured with current positional data, and then used by a steering accumulator. The component selector is essential for constraining movement.

```java
// Standard usage within a hypothetical NPC controller
Steering output = new Steering();
Vector3d npcPosition = npc.getPosition();
Vector3d playerPosition = player.getPosition();

// Constrain movement to the XZ plane
Vector3d groundPlaneSelector = new Vector3d(1, 0, 1);

// Use a concrete subclass like SeekForce
SteeringForce seek = new SeekForce();
seek.setPositions(npcPosition, playerPosition);
seek.setComponentSelector(groundPlaneSelector);

seek.compute(output); // The accumulator 'output' is now modified
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse an instance for a new calculation without first calling `setPositions`. Doing so will result in the NPC using stale positional data from the previous tick.
- **Null Component Selector:** Failure to call `setComponentSelector` will result in a `NullPointerException` when `compute` is invoked. Every force must have an explicit component selector.
- **Misaligned Lifecycles:** Do not store an instance of this class as a long-term member of an NPC. Its state becomes invalid after a single tick. It should be treated as a temporary, per-tick calculator.

## Data Pipeline
This class acts as a stateful processor within the broader NPC steering pipeline. It consumes positional data and prepares it for a specific behavioral calculation.

> Flow:
> Entity State (Self Position) -> Targeting System (Target Position) -> **SteeringForceWithTarget** -> Concrete Force (e.g., SeekForce) -> Steering Accumulator -> Physics Velocity Update

