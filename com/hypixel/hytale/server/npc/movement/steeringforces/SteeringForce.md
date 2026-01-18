---
description: Architectural reference for the SteeringForce interface, the core contract for NPC movement behaviors.
---

# SteeringForce

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Strategy Interface

## Definition
```java
// Signature
interface SteeringForce {
   boolean compute(Steering var1);
}
```

## Architecture & Concepts
The SteeringForce interface defines the fundamental contract for all individual steering behaviors within the server-side NPC movement system. It is a core component of a **Strategy Pattern**, designed to decouple the high-level movement aggregator (the Steering object) from the low-level algorithms that calculate specific movement vectors.

Each implementation of SteeringForce represents a single, atomic "desire" or "force" acting upon an NPC. Examples include seeking a target, fleeing from a threat, or wandering aimlessly. The central Steering system iterates over a collection of active SteeringForce instances for a given NPC, invoking their `compute` method to incrementally build a final, combined movement vector for that game tick.

This design promotes modularity and extensibility. New movement behaviors can be added by simply creating a new class that implements this interface, without modifying the core NPC or Steering classes.

## Lifecycle & Ownership
As an interface, SteeringForce itself has no lifecycle. The following applies to its concrete implementations.

-   **Creation:** Implementations are typically instantiated by higher-level AI systems, such as a Behavior Tree, a Goal-Oriented Action Planner (GOAP), or a finite-state machine. They may be created once and reused if stateless, or created per-NPC if they contain state specific to that NPC's current task.
-   **Scope:** The lifetime of a SteeringForce implementation is tied to the AI component that owns it. A `SeekPlayer` force might exist only as long as the NPC is in an "aggressive" state, while a `AvoidObstacles` force may persist for the entire life of the NPC.
-   **Destruction:** Cleanup is managed by the owning AI system. When an AI state terminates, it is responsible for discarding any SteeringForce instances associated with that state.

## Internal State & Concurrency
-   **State:** The interface contract is stateless. However, concrete implementations may be stateful. For example, a `Wander` force might need to store internal state to produce smooth, non-erratic direction changes over time. Stateless implementations (e.g., a simple `Flee` force) are highly encouraged as they are reusable and carry less overhead.
-   **Thread Safety:** **Not thread-safe.** Implementations of SteeringForce are expected to be executed exclusively on the main server thread during the NPC update tick. The `compute` method directly mutates the passed-in Steering object, making concurrent access from multiple threads inherently unsafe and a source of severe race conditions. Do not invoke this from asynchronous tasks.

## API Surface
The public contract consists of a single method, which forms the entirety of the component's interaction surface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compute(Steering steering) | boolean | Varies | Calculates a steering vector and applies it to the mutable Steering object. Returns true if the force was applied, false otherwise. |

## Integration Patterns

### Standard Usage
A controlling system, such as an AI behavior, holds a list of active forces. During its update tick, it iterates through them, passing the same Steering object to each `compute` call to accumulate the final desired velocity.

```java
// Simplified example from a hypothetical AI controller
Steering steeringContext = npc.getSteering();
steeringContext.reset(); // Clear forces from previous tick

for (SteeringForce force : activeForces) {
    force.compute(steeringContext);
}

// The steeringContext now contains the combined result
npc.applySteering(steeringContext);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Instance Sharing:** Do not share a single *stateful* instance of a SteeringForce implementation across multiple NPCs. This will cause unpredictable behavior as NPCs overwrite each other's internal state (e.g., one NPC's wander calculation affecting another's).
-   **Incorrect Invocation Order:** The order in which forces are computed can be critical, especially when using weighting or prioritization. Applying a high-priority `AvoidCollision` force after a low-priority `Seek` force is standard practice. Reversing this order may lead to unintended movement.

## Data Pipeline
The interface acts as a processor in the NPC movement data flow. It consumes the NPC's current state and produces a directional force.

> Flow:
> NPC State (Position, Velocity, Target) -> **SteeringForce.compute()** -> Mutated Steering Object -> Movement Aggregator -> Final Velocity Calculation -> Physics System

