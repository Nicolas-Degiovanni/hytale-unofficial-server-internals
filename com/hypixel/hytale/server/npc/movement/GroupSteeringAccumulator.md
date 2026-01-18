---
description: Architectural reference for GroupSteeringAccumulator
---

# GroupSteeringAccumulator

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Transient

## Definition
```java
// Signature
public class GroupSteeringAccumulator {
```

## Architecture & Concepts

The GroupSteeringAccumulator is a stateful, short-lived calculator used to aggregate spatial and kinematic data from a group of entities relative to a central observer. It is a fundamental building block for implementing complex NPC group behaviors, such as flocking, swarming, and collision avoidance, often referred to as "steering behaviors" or Boids.

This class is not a persistent system or manager; it is a computational tool. Its core design follows the **Accumulator Pattern**:
1.  **Initialize:** The `begin` method resets the internal state and establishes the point of view (the position and view direction of the NPC performing the calculation).
2.  **Accumulate:** The `processEntity` method is called repeatedly within a loop for each neighboring entity. It filters neighbors based on range and view cone, then adds their properties (position, velocity, distance) to internal sum vectors.
3.  **Finalize:** The `end` method computes the final averages from the accumulated sums, making the results available for consumption.

The accumulator provides the raw data necessary to compute the three classic flocking rules:
*   **Cohesion:** The average position of the group (derived from `getSumOfPositions`).
*   **Alignment:** The average velocity of the group (derived from `getSumOfVelocities`).
*   **Separation:** The average distance vector from neighbors (derived from `getSumOfDistances`), used to steer away from them.

It operates directly on the server's Entity Component System data, querying components like TransformComponent and Velocity via a ComponentAccessor.

## Lifecycle & Ownership

-   **Creation:** An instance of GroupSteeringAccumulator is typically created on the stack or as a reused member variable within a higher-level steering or AI system. It is instantiated on-demand when an NPC needs to evaluate its local environment.
-   **Scope:** The object's lifetime is extremely brief, intended to exist only for the duration of a single NPC's AI tick within a single server frame. Its internal state is only valid between a `begin` and `end` call.
-   **Destruction:** The object is expected to be garbage collected immediately after its results are used. If it is a pooled or reused object, it **must** be reset via the `begin` method before any new accumulation cycle.

## Internal State & Concurrency

-   **State:** This class is highly mutable by design. Its primary function is to modify its internal sum vectors and count during the accumulation phase. The state is transient and should be considered invalid outside of a `begin`/`end` block.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access within the server's main update loop. Any concurrent invocation of its methods, especially `processEntity`, will result in race conditions and corrupt the final calculation. No synchronization mechanisms are used.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| begin(...) | void | O(1) | Resets all internal sums and sets the observer's position and view direction. Must be called before any processing. |
| processEntity(...) | void | O(1) | Processes a single entity. Adds its data to the sums if it is within the configured range and view cone. |
| end() | void | O(1) | Finalizes the calculation by averaging the accumulated sums. Must be called after all entities are processed. |
| setMaxRange(maxRange) | void | O(1) | Configures the maximum distance at which entities are considered for accumulation. |
| setViewConeHalfAngleCosine(cosine) | void | O(1) | Configures the view cone to filter entities that are behind the observer. |
| getSumOfVelocities() | Vector3d | O(1) | Returns the average velocity of the processed group. **Warning:** Only valid after `end()` is called. |
| getSumOfPositions() | Vector3d | O(1) | Returns the average position of the processed group. **Warning:** Only valid after `end()` is called. |
| getSumOfDistances() | Vector3d | O(1) | Returns the average displacement vector from the observer to the group. **Warning:** Only valid after `end()` is called. |
| getCount() | int | O(1) | Returns the number of entities that were successfully processed and included in the sums. |

## Integration Patterns

### Standard Usage

The class must be used in a strict `begin -> process -> end` sequence. This is typically done within an AI system that iterates over potential neighbors of an NPC.

```java
// In a hypothetical NPCAISystem...
GroupSteeringAccumulator accumulator = new GroupSteeringAccumulator();
accumulator.setMaxRange(16.0);

// 1. Initialize for the current NPC
accumulator.begin(npcRef, componentAccessor);

// 2. Accumulate data from neighbors
for (Ref<EntityStore> neighborRef : findNearbyEntities(npcRef)) {
    if (neighborRef.equals(npcRef)) continue;
    accumulator.processEntity(neighborRef, componentAccessor);
}

// 3. Finalize the calculation
accumulator.end();

// 4. Use the results to apply steering forces
if (accumulator.getCount() > 0) {
    Vector3d cohesionVector = accumulator.getSumOfPositions();
    Vector3d alignmentVector = accumulator.getSumOfVelocities();
    // ... apply forces to the NPC's velocity component
}
```

### Anti-Patterns (Do NOT do this)

-   **Reusing Without Resetting:** Reusing an accumulator instance for a new NPC without first calling `begin` will corrupt the new calculation with stale data from the previous one.
-   **Reading State Mid-Cycle:** Accessing the result vectors (e.g., `getSumOfVelocities`) before `end` has been called will yield incomplete, un-averaged data. The values are only meaningful after finalization.
-   **Shared Instances:** Do not share a single accumulator instance across different AI updates that could run concurrently or out of sequence. Each calculation requires its own isolated instance or a properly reset one.

## Data Pipeline

The GroupSteeringAccumulator acts as a processor within the NPC AI data flow. It does not source or sink data permanently but transforms a set of entity states into a concise summary for decision-making.

> Flow:
> AI System Tick -> Query for Nearby Entities -> **GroupSteeringAccumulator.begin()** -> Loop: **GroupSteeringAccumulator.processEntity()** -> **GroupSteeringAccumulator.end()** -> Steering Behavior Logic -> Apply Force to NPC Velocity Component

