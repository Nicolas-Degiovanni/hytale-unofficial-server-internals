---
description: Architectural reference for PositionProbeAir
---

# PositionProbeAir

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient Utility

## Definition
```java
// Signature
public class PositionProbeAir extends PositionProbeBase {
```

## Architecture & Concepts
PositionProbeAir is a specialized, stateful utility for environmental queries within the server's physics and AI engine. It extends the generic PositionProbeBase to answer a specific question: is a given volume of space occupied primarily by air, or is it obstructed by solid or fluid blocks?

Its primary role is to support NPC and entity behavior, such as pathfinding, jump validation, and fall detection. It determines if an entity has solid ground beneath it or if it is currently airborne.

The class implements a form of the **Strategy Pattern**. The generic block iteration logic resides in the parent PositionProbeBase. PositionProbeAir provides a specific evaluation strategy via its private blockTest method, which is passed as a method reference to the parent's core probing algorithm. This design allows for a highly reusable core collision loop while enabling specialized probes for different environmental conditions (e.g., checking for air, water, or climbable surfaces).

## Lifecycle & Ownership
- **Creation:** PositionProbeAir is designed for ephemeral use. It is instantiated directly by a client system, such as an NPC behavior tree or a physics update task, immediately before a query is needed. It is not managed by a central registry or factory.

- **Scope:** The object's lifecycle is extremely short, typically confined to the scope of a single method call. An instance is created, the probePosition method is invoked, the results are read, and the object is then discarded.

- **Destruction:** The instance becomes eligible for garbage collection as soon as it falls out of scope. The presence of a reset method implies that object pooling is a potential optimization, but the typical use case is direct, short-lived instantiation.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its core purpose is to accumulate state during the execution of the probePosition method. Fields such as inAir, onSolid, onGround, and inWater are modified as the probe intersects with world geometry. The state of a PositionProbeAir instance is only considered valid and complete *after* a call to probePosition has returned.

- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access. Sharing an instance across multiple threads or attempting to read its state while a probe is in progress will result in race conditions and undefined behavior. Each thread or distinct logical operation must use its own private instance.

## API Surface
The public API is minimal, focusing on executing the probe and retrieving the results.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| probePosition(...) | boolean | O(N) | Executes the environmental query. N is the number of blocks intersected by the bounding box. Mutates internal state and returns the final inAir status. |
| isInAir() | boolean | O(1) | Returns true if the probe concluded the volume is in the air. Must be called after probePosition. |
| isOnSolid() | boolean | O(1) | Returns true if the probe detected solid ground during the query. Must be called after probePosition. |

## Integration Patterns

### Standard Usage
A higher-level system, such as an NPC's movement controller, instantiates the probe to check conditions before executing an action like jumping.

```java
// Hypothetical NPC jump logic
void attemptJump(Entity npc) {
    // Create a new probe for this specific check
    PositionProbeAir airProbe = new PositionProbeAir();
    CollisionResult result = new CollisionResult();
    
    // Configure the collision check (e.g., only collide with solid blocks)
    result.getConfig().setCollisionByMaterial(CollisionFlags.SOLID);

    // Execute the probe at the NPC's current position
    airProbe.probePosition(
        world.getEntityStoreRef(),
        npc.getBoundingBox(),
        npc.getPosition(),
        result,
        world.getComponentAccessor()
    );

    // Read the results to make a decision
    if (airProbe.isOnSolid()) {
        // It's safe to jump because we are on solid ground
        npc.applyJumpForce();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not share a single PositionProbeAir instance across different entities or for different ticks. The internal state is specific to a single query and will be corrupted if used for multiple, unrelated checks. Always create a new instance for each distinct environmental query.

- **Premature State Reading:** Accessing result methods like isInAir or isOnSolid *before* calling probePosition will yield meaningless default values (false). The object's state is only valid after the probe has been executed.

- **Ignoring the Base Class State:** This probe inherits state from PositionProbeBase, such as onGround and inWater. A consumer of this class may need to inspect the full state from both the child and parent class for a complete picture of the environment.

## Data Pipeline
The flow of data for a single query is a sequence of delegation and state mutation.

> Flow:
> 1. **Client System** (e.g., NPC AI) instantiates **PositionProbeAir**.
> 2. Client calls **probePosition**, providing world data, a bounding box, and a position.
> 3. The call is delegated to **PositionProbeBase**, which begins iterating over intersecting world blocks.
> 4. For each block intersection, PositionProbeBase invokes the **PositionProbeAir.blockTest** method reference.
> 5. **blockTest** evaluates the block's material and geometry, mutating the internal state fields (onSolid, onGround, inWater, etc.) of the PositionProbeAir instance.
> 6. After all blocks are tested, control returns to the **Client System**.
> 7. The Client System reads the final, aggregated state by calling **isInAir()** and **isOnSolid()**.

