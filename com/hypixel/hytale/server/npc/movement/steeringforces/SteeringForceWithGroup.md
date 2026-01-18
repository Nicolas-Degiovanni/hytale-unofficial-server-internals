---
description: Architectural reference for SteeringForceWithGroup
---

# SteeringForceWithGroup

**Package:** com.hypixel.hytale.server.npc.movement.steeringforces
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public abstract class SteeringForceWithGroup implements SteeringForce {
```

## Architecture & Concepts
SteeringForceWithGroup is an abstract base class within the server-side NPC movement AI framework. It serves as the foundation for all steering behaviors that derive their influence from a *group* of entities, rather than a single target or environmental factor. It embodies the "aggregate and compute" pattern, where information from multiple sources is collected before a final steering vector is calculated.

This class is a specialization of the SteeringForce interface, designed specifically for common group behaviors like flocking, collision avoidance, and cohesion. Concrete implementations (e.g., a SeparationForce or an AlignmentForce) extend this class to implement the specific logic for how each neighboring entity contributes to the overall force.

Its primary role is to provide a standardized contract and shared state (like the NPC's own position) for these group-based calculations, which are performed on a per-tick basis by the NPC's AI controller.

### Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created on-demand by higher-level AI behaviors, such as a FlockingSystem or a CombatAIController, at the beginning of a movement calculation cycle. They are not managed by a central registry or service locator.
- **Scope:** The lifecycle of a SteeringForceWithGroup instance is extremely short, typically confined to a single game tick for a single NPC. It is created, used for one calculation, and then becomes eligible for garbage collection.
- **Destruction:** The object is not explicitly destroyed. It is abandoned after the `compute` method has been called and its resulting steering vector has been consumed by the movement system. The Java Garbage Collector is responsible for reclaiming its memory.

## Internal State & Concurrency
- **State:** Highly mutable. The class maintains internal state such as `selfPosition` and any accumulated values within its concrete subclasses. The `reset` method is a critical part of its contract, designed to clear this state before a new calculation begins.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be used as a temporary, single-threaded calculator object within the context of a specific NPC's AI update tick. Concurrent calls to `add` or `reset` would lead to severe data corruption and unpredictable AI behavior.

## API Surface
The public API provides the necessary hooks to initialize, populate, and execute the force calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setSelf(ref, position, accessor) | void | O(1) | Initializes the force with the position of the NPC being controlled. Must be called before `add`. |
| setComponentSelector(selector) | void | O(1) | Configures an optional filter, likely used to select specific components from entities being considered. |
| reset() | void | O(1) | Clears all accumulated internal state. **CRITICAL:** Must be called before starting a new group calculation. |
| add(ref, commandBuffer) | void | O(N) | Adds a neighboring entity to the calculation. This method is called iteratively for each entity in the group. Complexity depends on subclass implementation. |
| compute(output) | boolean | O(N) | Calculates the final steering vector based on all entities added since the last `reset`. The base implementation is a no-op; subclasses must provide the core logic. |

## Integration Patterns

### Standard Usage
A controlling system (e.g., a FlockingBehavior) instantiates a concrete force, resets it, iterates through a list of nearby entities to populate it, and finally computes the result.

```java
// Assume separationForce is a concrete subclass of SteeringForceWithGroup
// This logic would exist within a higher-level AI system.

Steering output = new Steering();
List<Ref<EntityStore>> neighbors = findNearbyEntities(self);

// 1. Initialize the force calculator
separationForce.reset();
separationForce.setSelf(selfRef, selfPosition, componentAccessor);

// 2. Add all relevant neighbors to the calculation
for (Ref<EntityStore> neighbor : neighbors) {
    separationForce.add(neighbor, commandBuffer);
}

// 3. Compute the final steering vector
boolean hasForce = separationForce.compute(output);

if (hasForce) {
    // Apply the output.linear vector to the NPC's velocity
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use Without Reset:** Failing to call `reset()` before a new calculation cycle will cause data from the previous tick to corrupt the current one, resulting in erratic and unpredictable movement.
- **Shared Instances:** Do not share a single instance of a SteeringForceWithGroup subclass across multiple NPCs. Each NPC must have its own dedicated instance for the duration of its AI tick to prevent state corruption.
- **Incorrect Call Order:** Calling `compute` before any entities have been passed to `add` will result in a zero-vector or otherwise meaningless output. The `setSelf` method must also be called before any calls to `add`.

## Data Pipeline
This class acts as a stateful aggregator within the AI data pipeline. It transforms a collection of entity data into a single, actionable steering vector.

> Flow:
> AI Tick Start -> World Query (find nearby entities) -> **SteeringForceWithGroup.add()** (called for each entity) -> **SteeringForceWithGroup.compute()** -> Steering Vector -> Movement System -> Physics Update

