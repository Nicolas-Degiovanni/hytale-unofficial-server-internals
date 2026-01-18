---
description: Architectural reference for SensorReadPosition
---

# SensorReadPosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorReadPosition extends SensorBase {
```

## Architecture & Concepts
SensorReadPosition is a fundamental component within the Hytale NPC AI framework, operating as a spatial predicate. Its primary function is to determine if an NPC is within a specified distance range of a target coordinate. This class embodies the "sense" part of a standard sense-act AI cycle.

Unlike sensors that track live entities directly, SensorReadPosition operates on *stored positional data*. This data is retrieved from the NPC's memory system, specifically the MarkedEntitySupport, which acts as a blackboard for storing references and coordinates.

The component can operate in two distinct modes:
1.  **Marked Target Mode:** It resolves a stored entity reference (identified by an integer slot), retrieves that entity's live TransformComponent, and uses its current position as the target.
2.  **Stored Position Mode:** It directly retrieves a Vector3d coordinate previously stored in the same memory slot.

The result of a successful match—the target position—is not returned directly. Instead, it is cached within an internal PositionProvider. Other AI components, such as pathfinding or targeting actuators, are expected to query this provider via the getSensorInfo method to retrieve the data. This pattern decouples the act of sensing from the act of decision-making.

## Lifecycle & Ownership
-   **Creation:** SensorReadPosition is not instantiated directly. It is constructed during the server's asset loading phase via its corresponding builder, BuilderSensorReadPosition. Its configuration (range, slot, mode) is defined declaratively in NPC asset files.
-   **Scope:** The object's lifetime is tightly bound to the NPC entity that owns it. It is created when the NPC is spawned with the corresponding AI definition and persists for the entire life of that NPC.
-   **Destruction:** The object is eligible for garbage collection when its parent NPC entity is despawned or destroyed. It has no explicit cleanup or destruction method.

## Internal State & Concurrency
-   **State:** This class is stateful. It holds immutable configuration parameters set at creation time (slot, useMarkedTarget, minRange, range). Critically, it also manages the mutable state of its internal PositionProvider. The matches method modifies this provider, either by setting a valid target position or by clearing it.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be operated exclusively by the single thread responsible for its parent NPC's AI update tick. The matches method performs a non-atomic read-modify-write operation on the internal positionProvider. Concurrent access would lead to race conditions and unpredictable AI behavior.

    **WARNING:** Never access an instance of this class from any thread other than the main server thread or the designated world-update thread.

## API Surface
The public contract is minimal, focusing on the core sense-and-provide pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor's condition. Returns true if the NPC is within the configured range of the target position. This method has side effects, updating the internal PositionProvider. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal PositionProvider, which acts as a handle to the result of the last successful matches call. |

## Integration Patterns

### Standard Usage
This sensor is intended to be used as a condition within a larger AI structure, such as a Behavior Tree or a Finite State Machine. The controlling logic calls matches on each tick. If it returns true, subsequent actions can safely retrieve the target position from the InfoProvider.

```java
// Executed within an NPC's AI update loop
SensorReadPosition sensor = npc.getBehaviorComponent(SensorReadPosition.class);
Store<EntityStore> entityStore = world.getEntityStore();

// First, run the sensor to update its internal state
boolean inRange = sensor.matches(npc.getRef(), npc.getRole(), dt, entityStore);

if (inRange) {
    // If the sensor matched, retrieve the data
    PositionProvider provider = (PositionProvider) sensor.getSensorInfo();
    Vector3d target = provider.getTarget();

    // Now, use the target position for another system (e.g., pathfinding)
    npc.getPathfinder().setTarget(target);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Desynchronization:** Calling getSensorInfo and using its data without first calling matches in the same update tick. The provider's data may be stale from a previous tick or invalid if the last match failed.
-   **Ignoring Return Value:** Assuming a successful match and accessing the PositionProvider regardless of the boolean result from the matches method. If matches returns false, the provider is explicitly cleared, and attempting to use its data will result in errors or incorrect behavior.
-   **Builder Misconfiguration:** Configuring the sensor with an invalid slot index or setting useMarkedTarget to true when the slot is intended to hold a raw position can lead to runtime assertion errors or persistent sensor failure.

## Data Pipeline
The flow of data through this component is linear and triggered by each AI tick.

> Flow:
> 1. NPC MarkedEntitySupport (provides Entity Ref or Vector3d)
> 2. EntityStore (resolves Entity Ref to TransformComponent)
> 3. **matches()** (calculates distance)
> 4. Internal PositionProvider (state is updated with target Vector3d or cleared)
> 5. **getSensorInfo()** (exposes PositionProvider to consumers)
> 6. AI Actuator (reads position from provider to execute a task)

